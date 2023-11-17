---
date:  Thursday, November 16, 2023
tags:
---

# Email setup of some guy

[original](https://adorablefilthyorder--five-nine.repl.co/)

email-setup.md

 My Email Setup

This is a note to share my email setup. I use a Linux machine, so this is most
useful for people who work on Linux and prefer terminal mode.

The features of my setup are as follows

  • Local mail storage for speed. Maildir is the format or mail storage.

  • two way syncing between mail server and local storage. This is based on
    mbsync

  • mutt as the mail client. I use vim to compose and reply emails.

  • msmtp is used to send emails. I also setup a queue, so that all out-going
    emails are first put in the queue, and are sent out after a 5-minute delay.
    It is possible to force immediate send. This is adapted from a msmtpq
    script that was floating on the internet.

  • I use Gmail, with XOAUTH2 authentication. The authentication is necessary
    because my work account uses G-Suite, and password based authentication is
    disabled. Other IMAP/SMTP servers should also work.

  • mail searching is based on mairix. It is fast, and has a very small
    footprint. It serves my needs.

  • Mail filtering is based on maildrop from Courier mail. I maintain two
    lists, a list of emalis that are blocked, and a list of emails that are of
    the notification type. Emails from blocked addresses are sent to a
    Junk-Emails folder. Emails from notification addresses are sent to a
    notifications folder. I check the notifications folder once in a while
    only. There are mutt shortcuts that add the From address of the current
    email to the lists. Customized rules can be added to the maildrop filter
    rules.

There are some other convenience features. I will try to be as comphrehensive
as possible. Code snippets will be included. I think it is clearer to share
only the relevant parts rather than dump the whole configurations.

 General setup

The emails are stored on the server, and synced with local storage using mbsync
via the IMAPS protocol. Emails are sent using msmtp using SMTP protocol. So it
is like:

Email Server <-- mbsync ---> Local Emails ---> mutt

Email Server <-- msmtp <--- mail-send <--- mutt-check-attachment <-- mutt

 mbsync setup

mbsync configuration is given below.

IMAPAccount gmail
Host imap.gmail.com
User username@gmail.com
AuthMechs XOAUTH2
PassCmd "gmail-xoauth-tokens gmail"
SSLType IMAPS
CertificateFile /etc/ssl/certs/ca-certificates.crt

IMAPStore gmail-remote
Account gmail

SyncState *
Expunge Both
Create Slave
CopyArrivalDate yes

MaildirStore gmail-local
Path ~/Mail/Gmail/
Inbox ~/Mail/Gmail/INBOX

Channel INBOX
Master :gmail-remote:INBOX
Slave :gmail-local:INBOX

Channel sent
Master :gmail-remote:[Gmail]/"Sent Mail"
Slave :gmail-local:Sent-Items-Server
Sync Pull
MaxMessages 100

Channel drafts
Master :gmail-remote:[Gmail]/Drafts
Slave :gmail-local:Drafts

# Channel all
# Master :gmail-remote:[Gmail]/"All Mail"
# Slave :gmail-local:All-Mail
# MaxMessages 1000
# ExpireUnread yes

Channel other-folders
Master :gmail-remote:
Slave :gmail-local:
Patterns notifications Junk-Email Hold old-hold
Patterns Todo Deleted-Items

Note that I don't sync the "All Mail" folder. I want to use Gmail as a regular
IMAP server as any other IMAP server. Deleted emails are sent to the
Deleted-Items folder, which is merely a label on Gmail server. This is
different from the Trash folder on Gmail -- emails in Trash will be deleted
permanently after 30 days. I want to keep all my emails on the server. I have a
mutt shortcut h that puts an email in the Hold folder. These are emails that
can be periodically purged (say only keep 6 months of worth).

Check [https://wiki.archlinux.org/index.php/Isync#Using_XOAUTH2] on how to use
the XOAUTH2. Note that an XOAUTH2 SASL plugin is needed.

The script gmail-xoauth-tokens is a script that I wrote that will generate the
access token for a given account -- gmail in the example above.

 msmtp setting

My msmtp setting is as follows:

defaults

account Gmail
host smtp.gmail.com
port 587
from username@gmail.com
tls on
tls_starttls on
tls_certcheck off
auth oauthbearer
user username@domain.com
passwordeval "gmail-xoauth-tokens gmail"
logfile ~/soft/msmtp/msmtp.log

Note the oauthbearer authentication.

I put this in a file ~/soft/msmtp/msmtprc and msmtp is called by another script
mail-send, which implements queuing feature. The following is the mail-send
script.

#!/usr/bin/env bash

#  mail-send: based on Martin Lambers and Chris Gianniotis' msmtpq
#  Features:
#     1. Uses per message lock files.
#     2. Message to sent is always queued to prevent lost message
#     3. single entry point with different parameters for functionalities

usage() {
  message ''\
      'usage : mail-send '\
      '          -queue {msmtp arguments}  put a mail in the queue'\
      '        -sendnow {msmtp arguments} queue and immediately send out'\
      '              -f   flush all mail in queue'\
      '              -o   flush old mail in queue'\
      '              -R   select mail to send '\
      '              -p   select mail to purge'\
      '              -d   display queue'\
      '              -h   this helpful blurt' ''
}

settings(){
  # global settings
  unalias -a
  buffer_time=300 # always buffer messages for this many seconds before sending

  Q=~/soft/mail-send
  [ -d "$Q" ] || mkdir -p $Q

  LOG=~/soft/log/msmtp.queue-log

  sink-send(){
    return 0
  }

  case $HOSTNAME in
     test) MSMTP="sink-send" ;;
        *) MSMTP="/usr/bin/msmtp -C $HOME/soft/msmtp/msmtprc" ;;
  esac

  on_exit() {
    for file in $all_temp_files ; do
      if [ -f $file ]; then
        rm $file
      fi
    done
  }

  trap on_exit EXIT
}

message() { local L ; for L ; do [ -n "$L" ] && echo "  $L" || echo ; done ; }

log() {
  # usage : log [ -e errcode ] msg [ msg ... ]
  local line RC prefix="$('date' +'%Y %d %b %H:%M:%S')"
  if [ "$1" = '-e' ] ; then # exit error code
    RC="$2"
    shift 2
  fi
  message "$@"
  if [ -n "$LOG" ] ; then
    for line ; do
      [ -n "$line" ] && \
        echo "$prefix : $line" >> "$LOG"
    done
  fi
  if [ -n "$RC" ] ; then
    [ -n "$LOG" ] && \
      echo "    exit code = $RC" >> "$LOG"
    exit $RC
  fi
}

get-ok() {
  ## get user [y/n] acknowledgement
  local R YN='Y/n'          # default to yes

  [ "$1" = '-n' ] && \
    { YN='y/N' ; shift ; }  # default to no

  echo -n "$@" " "
  while true ; do
    echo -en "[${YN}]: " ; read R
    case $R in
      y|Y) return 0 ;;
      n|N) return 1 ;;
      '')  [ "$YN" = 'Y/n' ] && return 0 || return 1 ;;
      *)   echo 'yYnN<cr> only please' ;;
    esac
  done
}

purge-mail::remove-from-sent::get-message-id(){
  ## get message id from a maildir file
  gawk '/^Message-ID:/ {gsub(/[<>]/,""); print $2; exit}' "$1"
}

purge-mail::remove-from-sent(){ # ID MAILID
  # ID format 2013-05-30-10.15.24
  # Use ID to find files in $SENT/ folder, and confirm using MAILID for delete
  local SENT MAILID TIME ID file from account
  from=$(grep -h -m1 '^From: ' "$Q"/"$1".mail)
  case "$from" in
    *username@gmail.com*) account=Gmail;;
    *workname@work.com*) account=Work;;
  esac
  SENT=~/Mail/$account/Sent-Items/cur
  TRASH=~/Mail/$account/Deleted-Items/new/
  MAILID="$2"
  TIME=$(echo "$1" |sed -e \
    's/\([0-9]\{4\}\)-\([0-9]\{2\}\)-\([0-9]\{2\}\)-/\1-\2-\3 /' \
    -e 's/\./:/g')
  TIME=$(date +%s -d "$TIME")
  shopt -s nullglob
  for file in $SENT/$TIME*; do
    ID=$(purge-mail::remove-from-sent::get-message-id $file)
    if [ "$ID" = "$MAILID" ]; then
      id=$(date +%s).$(tr -dc 1-9 < /dev/urandom  2>/dev/null \
          | head -c5)_1.$(hostname)
      mv $file $TRASH/$id
      echo "  moved "$1" to Deleted Items"
    fi
  done
}

purge-mail(){
  # purge a single mail
  local ID=$1
  if lock-mail $ID; then
    MAILID=$(purge-mail::remove-from-sent::get-message-id "$Q/$ID".mail)
    purge-mail::remove-from-sent "$ID" "$MAILID"
    rm -f "$Q"/"$ID".*   # msmtp - nukes both files in queue
    log "mail [ $ID ] purged from queue"
    lock-mail -u $ID
  else
    message '' 'mail is locked -- cannot purge'
  fi
}

send-mail() {
  # send a single mail
  local ID=$1
  local FQP="${Q}/${ID}"
  local -i RC=0
  if ! lock-mail $ID; then
    message "mail $ID is locked"
    return 1
  fi
  if [ -f "${FQP}.msmtp" ] ; then    # corresponding .msmtp file found
    # verify net connection - ping ip address of debian.org
    if ping -4 -qnc1 -w2 debian.org > /dev/null 2>&1 ; then       # connected
      $MSMTP --read-envelope-from $(< "${FQP}.msmtp") < "${FQP}.mail"
      if [ $? -eq 0 ]; then # this mail goes out the door
        rm -f ${FQP}.*      # nuke both queue mail files
        log "mail [ $ID ] from queue ; send was successful ; purged from queue"
      else                           # send was unsuccessful
        RC=$?                        # take msmtp exit code
        log "mail [ $ID ] from queue ; send failed ; msmtp rc = $RC"
      fi
    else                             # not connected
      message "mail [ $ID ] from queue ; couldn't be sent - host not connected"
      RC=1
    fi
  else                               # corresponding MSF file not found
    RC=1
    log "preparing to send .mail file [ ${FQP}.mail ] but"\
        "  corresponding .msmtp file [ ${FQP}.msmtp ] was not found in queue"\
        '  skipping this mail ; this is worth looking into'
  fi
  lock-mail -u $ID
  return $RC
}

flush-queue() {
  # send all mails in the queue. If -o is provided, then only old emails
  local CMIN mail sent=N
  if [ "$1" = "-o" ]; then
    CMIN="-cmin +$(( $buffer_time / 60 ))"
  else
    CMIN=""
  fi
  while read -r mail; do
    mail=${mail%.mail}
    mail=${mail##*/}
    send-mail $mail
    sent=Y
  done < <(find $Q -name '*.mail' $CMIN -print)
  [ $sent = "N" ] && message '' 'mail queue is empty (nothing to send)' ''
}

display-queue() {
  ## display queue contents
  local M LST="$(ls $Q/*.mail 2>/dev/null)"
  if [ -n "$LST" ] ; then
    for M in $LST ; do
      message '' "mail id = [ $(basename $M .mail) ]"
      'egrep' -s --colour -h '(^From:|^To:|^Cc:|^Subject:)' "$M"
    done
    echo
  else
    message '' 'no mail in queue' ''
  fi
}

select-mail() {  # <-- '-purge' or '-send'
  ## select a single mail from queue, -purge it or -send it
  local M LST="$(ls $Q/*.mail 2>/dev/null)"
  if [ -n "$LST" ] ; then
    for M in $LST ; do
      local ID=$(basename $M .mail)
      message '' "mail id = [ $ID ]"            # show mail id
      egrep -s --colour -h '(^From:|^To:|^Subject:)' "$M"
      echo

      if get-ok "$1" ; then
        if [ "$1" = '-purge' ] ; then
          purge-mail $ID
        else
          send_mail $ID
        fi
      else  # user opts out
        message '' 'nothing done to this queued email' ''
      fi
    done
  else
    message '' 'no mail in queue' ''
  fi
}

lock-mail(){
  # lock a single mail with ID $1
  case "$1" in
    -u)
      id=$2
      /bin/rm -f $Q/$id.lock
      ;;
    *)
      local id=$1
      local tempfile=$(mktemp $Q/lock-${ID}-XXXXX)
      all_temp_files="$all_temp_files $tempfile" # save for trap to remove
      local lockfile=$Q/$id.lock
      if ln $tempfile $lockfile ; then
        return 0
      else
        rm $tempfile
        return -1
      fi
      ;;
  esac
}

queue-mail::make-id() {
  # make a unique ID for mail
  local -i INC
  ID="$(date +%Y-%m-%d-%H.%M.%S)"
  if [ -f "$Q/$ID.*" ] ; then # ensure fqp name is unique
    INC=1
          while [ -f "$Q/${ID}-${INC}.*" ] ; do
      (( ++INC ))
          done
          ID="${ID}-${INC}"
  fi
}

queue-mail(){
  # queue the mail from stdin
  queue-mail::make-id
  lock-mail $ID
  local FQP="$Q/$ID"
  cat > "${FQP}.mail" || \
    log -e "$?" "creating mail body file [ ${FQP}.mail ] : failed"
  if echo "$@" > "${FQP}.msmtp" ; then
    log "enqueued mail as : [ $ID ] ( $* ) : successful"
  else                                     # write failed ; bomb
    log -e "$?" "queueing - writing msmtp cmd line { $* }"\
                "           to [ ${ID}.msmtp ] : failed"
  fi
  lock-mail -u $ID
}

main(){
  settings
  case "$1" in
    -queue)
      shift 1
      queue-mail "$@"
      (
      # Trap necessary because subshell doesn't see/honor main-trap
      trap on_exit EXIT
      sleep $buffer_time
      send-mail "$ID" )&
      ;;
    -sendnow)
      shift 1
      queue-mail "$@"
      send-mail "$ID"
      ;;
    -d) display-queue ;;
    -f) flush-queue ;;
    -o) flush-queue -o ;; # flush only old mails
    -R) select-mail -send ;;
    -p) select-mail -purge ;;
    -h) usage ;;
  esac
  exit 0
}

main "$@"

Note that the email folders are hard-coded. Search for account=Gmail and
account=Work to change. Also the command MSMTP may need to be configured
depending on your setup.

 Mail searching

The mails can be searched within mutt, using a shortcut s, which is a macro
that is bound to a macro that calls a bash script to do the searching and
presenting the results back to mutt. The script is as follows.

#! /bin/sh -

# perform search for mutt.

# This script is called by mutt via ` ` so that its output
# is interpreted by mutt as input. So output to &1 will be sent
# to Mutt as input.

# If -S is provided, then perform the search using mairix.
# If -L is provided, then try to mimic the "limit" function
# If -l is provided, then do "limit" search, but with alias replacement
# If -V is provided, virtual folders are used

initialization(){
  result_folder=~/soft/mairix/search-result

  case "$1" in
    -l) mode="limit_search_term";;
    -V) mode="mairix_virtual_folders";;
    -L) mode="limit_virtual_folders";;
    -S) mode="mairix_search_term" ;;
  esac

  # account
  account=${2#-}

  # virtual folder list
  vf=~/.mutt/virtual-folders-$account
  mairixrc=~/soft/mairix/config/mairixrc-$account

  # disable globbing.
  set -f

  # restore stdin/stdout to the terminal, fd 3 goes to mutt's backticks.
  exec < /dev/tty 3>&1 > /dev/tty # see header comment

  # save tty settings before modifying them
  saved_tty_settings=$(stty -g)

  # upon SIGINT (mapped to <Ctrl-G> below instead of <Ctrl-C> to match
  # mutt behavior), cancel the action.
  trap '
      printf "\r"; tput ed; tput rc
      stty "$saved_tty_settings"
      exit
  ' INT TERM

  cmd=

  # put the terminal in cooked mode. Set eof to <Return> so that pressing
  # <Return> doesn't move the cursor to the next line. Disable <Ctrl-Z>
  #stty icanon echo ctlecho eof '^M' intr '^G' susp '' erase '^?'
  stty icanon echo echoe echok eof '^M' intr '^G' susp '' erase '^?'

  # save cursor position:
  tput sc
}

run_mairix(){
  # search the mails using mairix: first check the virtual folder list
  mairix -f $mairixrc $* > /dev/null
  # find $result_folder -type l | wc -l
}

get_ids(){
  # get ids for virtual folders
  folder="${1//[(\/\+)]/.}" # replace ()/ with .
  gawk -v mode=${mode:0:6} -v folder="$folder" '
    BEGIN{ PRE="f:";
      sep["limit_"]="|"; sep["mairix"]="/";
      prefix["limit_.from"]="~f<space>"; prefix["limit_.to"]="~t<space>";
      prefix["mairix.from"]=""; prefix["mairix.to"]="";

      }
    $0 ~ "^" folder {
      while(getline>0)
        if(/^  [a-zA-Z@]/){
          gsub(/ /,"")
          if(!/@/) gsub(/\\./,",")
          if(/^t:/){
            PRE="t:"; gsub(/^t:/,""); type=".to"
          }else
            type=".from"
          s=sep[mode]; p=prefix[mode type]
          result=result? result s p $0: p $0
        }else if(/F/){ # raw expressions
          s=sep[mode]
          gsub(/ /,"")
          result=result? result s $0: $0
        }else
          break
      }
    /^#END/{exit}
    END {
      if(mode=="limit")
        print result
      else
        print PRE result
      }' $vf
}

do_limit_search_term(){
  get_input "Limit to messages matching: "
  source ~/.mutt/alias-limit-search
  readarray -d " " arr <<<"$_input"
  new_input=""
  for str in ${arr[@]}; do
    if [ -v map_alias["$str"] ]; then
      new_input="$new_input ${map_alias[$str]}"
    else
      new_input="$new_input $str"
    fi
  done
  new_input=${new_input:1}
  cmd="<limit>${new_input// /<space>}<return>"
  printf "$cmd" >&3
}

do_mairix_virtual_folders(){
  # show a list of virtual folders, and ask user to select one
  folders=$(gawk '/^#END/{exit} /^[a-z0-9A-Z]/ { print }' $vf | sort)
  folder=$(menu "$folders")
  case $folder in
    q|x|"") return ;;
  esac
  result=$(get_ids ${mode:0:6} "$folder");
  [ -z "$result" ] && return
  cmd="<change-folder>$result_folder<return>"
  printf "$cmd" >&3
}

do_limit_virtual_folders(){
  # show a list of virtual folders, and ask user to select one
  folders=$(gawk '/^#END/{exit} /^[a-z0-9A-Z]/ { print }' $vf | sort)
  folder=$(menu "$folders")
  case $folder in
    q|x|"") return ;;
  esac
  result=$(get_ids ${mode:0:6} "$folder");
  [ -z "$result" ] && return
  cmd="<limit>$result<return>"
  printf "$cmd" >&3
}

do_mairix_search_term(){
  get_input "Pattern (-o for older mails): "
  case "$_input" in
    q) ;;
    *[.a-zA-Z0-9]*)
      if [[ "$search" =~ .*-o.* ]]; then
        run_mairix "${_input/-o/}" # want old mails as well
      else
        run_mairix "$_input d:1y-" # otherwise limit to one year
      fi
    ;;
  esac
  number=$(run_mairix $*);
  if [ "$number" -gt 500 ]; then
      echo -en "\rToo many ($number) matches. ";
      sleep 1
  else
    cmd="<change-folder>$result_folder<return>"
    printf "$cmd" >&3
  fi
}

get_input(){
  # get input using prompt in $1
  local prompt="$1"

  # go to last line of the screen and get input
  set $(stty size) # retrieve the size of the screen.
  tput cup "$1" 0
  tput ed
  tput sgr0
  printf "\r$prompt"
  _input=$(dd count=1 2> /dev/null)
}

main(){
  initialization "$@"

  do_$mode

  # clear our mess
  printf '\r'; tput ed

  printf "<refresh>" >&3

  # restore cursor position
  tput rc

  # and tty settings
  stty "$saved_tty_settings"
}

main "$@"


 Virtual folders

I use virtual folders as a nimble way to collect emails from a given set of
people as needed. The list of virtual folders are stored in a file called ~
/.mutt/virtual-folders-gmail, which has the following content:

# Virtual folder listing for mutt
#
# 1) lines that do not start with [A-Za-z ] are comments
# 2) Each folder name is followed by an indented list of email IDs (could be
#    full address. These IDs will be concatenated and used in mairix search)
# 3) If an email ID starts with t:, it is matched in To rather than From
# 4) Everything after the #END line are ignored
# 5) If t: is used, all addresses should be in such form
# 6) string= means substring match, which should be used for partial address
#    other than a single word

Expedia
  expediamail
  travel=

Committee
  scottaa
  alicebb
  tomccgreat
  judyfos

Thatconference
  address1@engx.bac.uk
  address2@gmailx.com
  username3@amazona.com

Education
  @university.edu
  support@instructure.com
  suzbarxy
# END
# furtuer lines are ignored

These virtual folders are used when a search is requested through mutt-mairix
script. The macros within mutt are defined as follows

set my_account=gmail
macro index,pager s ":set my_cmd=\`mutt-mairix -S -$my_account\`<return>\
:push \$my_cmd<return>"
macro index,pager L ":set my_cmd=\`mutt-mairix -L -$my_account\`<return>\
:push \$my_cmd<return>"
macro index,pager l ":set my_cmd=\`mutt-mairix -l -$my_account\`<return>\
:push \$my_cmd<return>"
macro index,pager V ":set my_cmd=\`mutt-mairix -V -$my_account\`<return>\
:push \$my_cmd<return>"

When s is pressed within mutt, the query will be prompted at the command line
of mutt, search will be done by mutt-mairix, and results displayed in mutt.

When L is pressed within mutt, a popup menu will be shown with the list of
virtual folders as defined in the virtual folder file, and selected entry will
be used to find the search terms, and then a limit search will be done on the
CURRENT folder, and results displayed in mutt.

When l is pressed, it will behave like the limit mode of mutt, with the
additional feature that words that are defined in a file ~/.mutt/
alias-limit-search are replaced by their replacement values. This is to deal
with names such as "Elizabeth Smith elsmith_999@gmailx.com", which may be
searched with an alias lizzy. The content of alias-limit-search is as follows:

#!/usr/bin/bash

# a bash file that will be sourced by mutt-mairix when doing limit search
# so that an alias is mapped to another search term for limit search in mutt

declare -A map_alias

add_map(){
  # add a pair of: search_term mapped_search_term
  let i=3
  while [ $i -le ${#1} ]; do
    map_alias[${1:0:$i}]=$2
    let i+=1
  done
}

add_map lizzy elsmith_999
add_map bob robert_smith@gmailx.com

The script is set up such that liz, lizz, lizzy are all transformed to
elsmith_999 (at least three characters are needed).

When V is pressed within mutt, a virtual-folder search will be performed on ALL
folders using mairix based on the selected entry from a pop-up menu, and the
results will be displayed in mutt.

The mutt-mairix script uses a helper script that shows a popup menu. The help
script is included below

#!/bin/bash

# give several ITEMS, display a menu, and ask the user to select a line
# from the ITEMS, and return the content of the line selected
# navigation keys supported:
#   j (down), k (up), m/M (middle), Enter/space(select)

# Example: echo -e " item1 \n item2 \n item3" | menu

declare -a ITEMS

# aux function
  puts(){
    tput cup $2 $1;
    [ "$4" ] && tput $4
    printf "$3"
  }

# Draw a box with upper left corner, nROWs, nCOLs
  draw_box(){
    # table chars
    # http://en.wikipedia.org/wiki/Box-drawing_character

    LR="\e(0j\e(B";
    UR="\e(0k\e(B";
    UL="\e(0l\e(B";
    LL="\e(0m\e(B";
    HH="\e(0q\e(B";
    VV="\e(0x\e(B";

    let NR=$nITEMS+2 NC=$textwidth+3
    puts $((X-2)) $((Y-1)) "$(printf %$((${NC}+6))s)"
    puts $((X-2)) $Y "  $UL\e(0"$(printf %${NC}s|tr ' ' 'q')"\e(B$UR  "
    for ((i=1; i<=$NR-2; i++)); do
      puts $((X-2)) $((Y+i)) "  $VV$(printf %${NC}s)$VV  "
    done
    puts $((X-2)) $((Y+NR-1)) "  $LL\e(0"$(printf %${NC}s|tr ' ' 'q')"\e(B$LR  "
    puts $((X-2)) $((Y+NR)) "$(printf %$((${NC}+6))s)"
  }

# print boxed text
  boxed_text(){
    set $(stty size)
    let Hsize=$2 Vsize=$1
    textwidth=$(let w=1; for i in ${!ITEMS[@]}; do
        [[ ${#ITEMS[$i]} -gt $w ]] && let w=${#ITEMS[$i]}
        done; echo $w)

    if [[ $textwidth -gt $((Hsize - 4)) ]] ; then
      let textwidth=Hsize-4

      for i in ${!ITEMS[@]}; do
        ITEMS[$i]=${ITEMS[$i]:0:${textwidth}-1}
      done
    fi

    let X=(Hsize-textwidth)/2 Y=(Vsize-nITEMS)/2
    puts $X $((Y-2)) "Please Select"
    draw_box

    for index in ${!ITEMS[@]} ; do
      item=${ITEMS[$index]}
      if test "$index" = "$cur" ; then
        puts $((X+2)) $((Y+index+1)) "$item" rev
      else
        puts $((X+2)) $((Y+index+1)) "$item" sgr0
      fi
    done
  }

# update display
  update_display(){
    if test $new -eq $cur ; then
      return
    else
      item=${ITEMS[$new]}
      # puts $((X+2)) $((Y+new)) "${item:0:$len}" rev
      puts $((X+2)) $((Y+new+1)) "$(printf %${#item}s)" rev
      puts $((X+2)) $((Y+new+1)) "$item" rev
      item=${ITEMS[$cur]}
      puts $((X+2)) $((Y+cur+1)) "$(printf %${#item}s)" sgr0
      puts $((X+2)) $((Y+cur+1)) "$item" sgr0
    fi
    let cur=new
  }

# Main
  main(){
    exec 3>&1 >/dev/tty </dev/tty
    tput civis
    readarray ITEMS <<< "$1"
    nITEMS=${#ITEMS[@]}
    let cur=0

    boxed_text

    IFS='' # preventing parsing input line
    while read -s -N 1 key; do
      key=${key,,} # change all to lower case
      case "$key" in
        q|$'\033')
          cur=1000
          break ;;
        j|" ")
          let new=cur+1; [[ $new -eq $nITEMS ]] && let new=nITEMS-1
          update_display ;;
        k)
          let new=cur-1; [[ $new -lt 0 ]] && let new=0
          update_display ;;
        [a-il-pr-z])  # these keys locate the entry with the first letter
          let new=cur i=0
          while [ $i -lt $nITEMS ] ; do
            local F=${ITEMS[$i]:0:1}
            F=${F,,}
            if [ $F = $key ] ; then
              let new=i
              break
            fi
            let i++
          done
          update_display ;;
        $'\x0a'|"")
          break
      esac
    done

    echo "${ITEMS[$cur]}" >&3
    tput cnorm
  }

if [ $# -eq 0 ]; then
  LINES=$(printf "A: Option1\nB:Option 2\nC:Option 3")
  main "$LINES"
else
  main "$1"
fi

 Mail filtering

I filter the incoming emails using maildrop. This works as follows: First,
mbsync is called to sync INBOX with the server. Then the following script is
run:

#!/bin/bash

# filter emails in a Maildir INBOX using mblaze commands

setup(){
  # setup INBOX and maildrop rules file
  case "$account" in
    gmail)
      INBOX=~/Mail/Gmail/INBOX
      rules=~/soft/maildrop/rules.gmail
      ;;
  esac
  seen_file=~/soft/log/email-filter-mblaze-$account-seen
  [ ! -f $seen_file ] && touch $seen_file
  SEEN=$(cat $seen_file)
}

process-message(){
  # process a single message file given in $1
  msg=$1
  base=$(basename $msg)
  base_with_flags=${base/,*}
  if [[ $SEEN == *"$base_with_flags"* ]]; then
    # echo already seen and processed
    return
  fi

  cat $msg | maildrop -m $rules |
  while read action parameter; do
    case $action in
      mflag) mflag $parameter $msg ;;
      mrefile) mrefile ${msg}* $parameter ;;
      xfilter) cat $msg | $parameter ;;
    esac
  done
  echo $msg >> $seen_file
}

usage(){
  echo "email-filter-mblaze {work|gmail}"
}

main(){
  if [ $# -eq 0 ]; then
    usage
    exit
  fi

  account=$1
  setup
  # list messages and process them one by one
  mlist -N $INBOX |
  while read msg; do
    process-message $msg
  done
}

main "$@"

# run: bash % gmail

The script needs some utilities from the https://github.com/leahneukirchen/
mblaze tool.

The script calls maildrop in the embedded mode -m and process the command
string that is output from maildrop. The maildrop rc file looks like the
following:

# Maildrop filter config file
#  Pattern matching is similar to Grep: 1) line based, 2) If left hand
#  side *contains* a substring that matches the right pattern, then success

import PATH
TYPE="maildir"
BLOCKED="$HOME/.mutt/blocked-addresses-gmail"
NOTIFY="$HOME/.mutt/notification-addresses-gmail"

# no logfile and DEFAULT for embedded mode
# logfile "$HOME/soft/maildrop/log.gmail"
# DEFAULT="$DIR/INBOX"
DIR="$HOME/Mail/Gmail"
notifications="$DIR/notifications"

if( /^From:\s*(.*)/ && lookup( $MATCH1, $NOTIFY) )
  {
    echo "mrefile $notifications"
    exit
  }

if( /^From:\s*(.*)/ && lookup( $MATCH1, $BLOCKED) )
  {
    echo "mrefile $DIR/Junk-Email"
    exit
  }

if ( /^From:.*mgnw13@univxxi.com/ && \
     /^Subject:.*(Pictures|Photos)/ )
  {
    echo "xfilter /home/shared/bin/email-save-pictures"
    exit
  }

if ( /^To:.*emailx02300@gmailx.com/ \
  ||/^To:.*mypapapy010@gmailx.com/ \
  )
  {
    echo "mrefile $DIR/Parents"
    exit
  }

 Blocked Addresses and Notification Addresses

I maintain two lists: one of blocked addresses and one of notification
addresses. Those in the first list will be blocked, and emails from those in
the second list are placed in the notifications folder --- which means they are
only for my information, and do not require any actions. These two files are
consulted by the maildrop filter setup above.

To add to these two lists, the following two shortcuts are included in mutt:

macro index,pager "\Cb" |mutt-block-address<space>-b<return>
macro index,pager "\Cn" |mutt-block-address<space>-n<return>

which will pipe the current message to the script mutt-block-address for
blocking or for notification. The script is as follows:

#!/bin/bash

# A filter that adds the From address of a message to the proper list

# mutt-block-address -b : send to blocked addresses
# mutt-block-address -n : send to notification addresses

# file of list of address to be blocked
BLOCKED=~/.mutt/blocked-addresses
NOTIFICATION=~/.mutt/notification-addresses
m=/dev/shm/tmp-$USER.msg

save-file(){
  cat > $m
}

get-to(){
  # get email To the message
  to=$(cat $m| sed '$!N;s/\n \+/ /;P;D' | grep -m 1 '^To: ')
  to=${to//To: /}
  to=${to//* /}
  to=${to//[<>]/}
  to=${to,,} # to lower case
}

set-message(){
  # $1 should be either -b or -n for Block or Notify
  case "$to" in
    *@gmail.com) account=gmail;;
    *) account=work;;
  esac

  NON="\e[0m"
  BLD="\[\e[1m\]"
  RED="\[\e[0;31m\]"
  WHT="\e[0;37m"

  case $1 in
    -b)
      folder=$BLOCKED-$account
      short_message="BLOCKED for $account: $WHT$email$NON"
      verb="Block"
      ;;
    -n)
      folder=$NOTIFICATION-$account
      short_message="NOTIFICATION for $account: $WHT$email$NON"
      verb="Set As Notify"
      ;;
    *)
      exit
  esac
}

get-from(){
  # get email from the message
  emailfull=$(cat $m | sed '$!N;s/\n \+/ /;P;D' | grep -m 1 '^From: ')
  emailfull=${emailfull//From: /}
  email=${emailfull//* /}
  email=${email//[<>]/}
}

confirm(){
  # tty-ask-YN "$verb $email ? (Y/N) "
  exec < /dev/tty 3>&1 > /dev/tty
  set $(stty size) # get size of screen
  tput sc #save cursor
  tput clear
  tput cup "$(( $1 / 2 ))" 0 # move to last line
  stty raw
  printf '\r%s %s (Y/N): ' "$verb" "$email"
  response=$(dd count=1 2> /dev/null)
  tput cuu1
  printf "%${COLUMNS}s" ""
  tput rc # restore cursor
  case "$response" in
    y|Y) return 0;;
    *)   return 1;;
  esac
}

add-to-list(){
  if ! grep -q -s $email $folder; then
    echo "$email" >> $folder
  fi
}

clean-up(){
  /bin/rm $m
}

feedback(){
  (
  exec < /dev/tty >/dev/tty
  set $(stty size) # get size of screen
  tput sc #save cursor
  tput cup "$1" 0 # move to last line
  echo -en "\r$short_message"
  # sleep 0.1
  # printf "\r%${COLUMNS}s" ""
  tput rc # restore cursor
  ) &
}

main(){
  save-file
  get-to
  get-from
  set-message $1
  confirm && add-to-list && feedback
  clean-up
}

main "$@"

Confirmation will be requested before actually placing the From address in the
lists.

 Email linking

I often find it necessary to refer to a particular email, say in a note file. I
do this in the following way. I have a shortcut y that will yank a URL like
mail:92a309 that can be pasted into any text file. This string can be clicked
on Urxvt terminal to bring back the specific mail pointed to by the string.
Roughly, how it works is the following. 1) the shortcut y will pipe the current
mutt message to a script. 2) The script will read the message-ID field from the
message. 3) The script then create a hash of the message-ID. It also saves the
hash value like 92a309 together with the full message-ID in a plain text file,
one line per message ID, such as

92a309 20820261987228.875087@gmailbb.xom.com

so that based on the hash value, a full message ID can be retrieved, and then
the mail can be found out through mairix search.

 Mairix setting

The mairix setting is pretty standard. I have three folders: ~/soft/mairix/
{config,database,search-result} and I place mairixrc-gmail in the ~/soft/mairix
/config folder with the following content:

base=/home/itsme
maildir=Mail/Gmail...
mfolder=~/soft/mairix/search-result
database=/home/itsme/soft/mairix/database/mairix-db-gmail

 Checking Attachments

All emails are checked if they are possibly missing attachments. This is done
by setting

set sendmail="/home/shared/bin/mutt-check-attachment"

in mutt. The script is given as follows:

#!/bin/bash

SENDMAIL="mail-send -queue"
LOG=$HOME/soft/log/mutt-check-attachment-log

if [ ! -f $LOG ]; then touch $LOG; fi

t=`mktemp -t mutt-attach.XXXXXX` || exit 1

cat > $t || exit 2

function ppid {
  echo $(ps -p $1 ho ppid)
}

function multipart {
    grep -q '^Content-Type: multipart' $t
}

function word-attach {
  gawk -- 'BEGIN{cont=0;}
    /^>/{next;}
    {$0=tolower($0)}
    /=$/{cont=1}
    /[^=]$/{cont=0}
    /attached|attachment/{if(cont) next; else exit 100}' $t
  if [ $? -eq 100 ]; then
    return 0
  else
    return 1
  fi
}

# see mutt-mairix script for more info on terminal usage
function ask {
    id=$(ppid $$)
    id=$(ppid $id) # this should be the mutt process
    tty=$(ps -p $id ho tty)
    exec < /dev/$tty 3>&1 >/dev/$tty
    saved_tty_settings=$(stty -g)
    trap '
        printf "\r"; tput ed; tput rc
        printf "<return>" >&3
        stty "$saved_tty_settings"
        exit
    ' INT TERM
    stty raw echo echoe eof '^M' intr '^G' susp '' erase '^?'
    set $(stty size)
    tput sc
    tput cup "$1" 0
    tput ed
    tput sgr0
    printf 'No attachment, continue (Y/N)? '

    response=$(dd count=1 bs=1 2>/dev/null)

    case $response in
      y|Y*)
        answer=0
        ;;
      *)
        answer=1
        ;;
    esac

    # clear our mess
    printf '\r'; tput ed

    # restore cursor position
    tput rc

    # and tty settings
    stty "$saved_tty_settings"
    return $answer
}

if multipart || ! word-attach || ask;  then
  (
    MQLOG=$HOME/soft/log/msmtp.queue-log
    if [ ! -f $MQLOG ]; then touch $MQLOG; fi
    size_before=$(stat -c %s $MQLOG)
    $SENDMAIL $@ < $t > /dev/null
    if [ ! $? -o \( $(stat -c %s $MQLOG) = $size_before \) ]; then
      zenity --warning --text  \
        "Message Send Failed. Check $HOME/soft/log/ and Sent-Items/
        $(head 20 $t)"
      echo -n "$(date): $0 Failed to deliever message to [ $@ ]\n$msg" >> $LOG
    fi
    /bin/rm $t
  )&
else
  # remove the Fcc file in SENT folder
  FROM=$(awk 'BEGIN{FS="<|>"} /^From:/{print $2; next}' $t)
  case "$FROM" in
    username@gmail.com) SENT=~/Mail/Gmail/Sent-Items ;;
    *) SENT=~/Mail/Work/Sent-Items ;;
  esac
  find $SENT -size $(stat -c '%s' $t)c -print | \
    while read file; do if cmp -s $t $file; then /bin/rm $file; fi; done
  /bin/rm $t
  exit 4
fi

Several scripts mentioned, including this one, can be optimized by using the
mblaze utilities. They were written before mblaze existed, and have not been
revised to take full advantage of mblaze. The ideas of the scripts are
definitely not mine (some maybe are). They were borrowed from some other online
scripts. I should include the original author name, but could not find them any
more. Now that I am sharing my files, in a hope to give back to the community.

