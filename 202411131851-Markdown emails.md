---
date:  2024-11-13
tags: mutt neomutt email markdown
---

# Markdown emails

## Original post 1

[Original](https://tom.wemyss.net/posts/neomutt-markdown-email/)

Writing emails in markdown with neomutt and pandoc

20 June 2020

Mockup showing HTML formatted email on iPhone and plaintext email in neomutt
The same email displayed in HTML on iPhone and in plaintext when viewed in
neomutt.

Mutt is a fast, user-friendly and easily-extensible terminal-based email
client. I’ve been using mutt for my emails for a couple of years, but I’ve
found myself falling back to using Google’s Gmail for emails that I cannot
easily compose in plain text - such as those that require text formatting. I
still really enjoy using mutt (there are a lot of reasons to prefer it over
other email clients), so I wanted to find a way to bring it back into my daily
email workflow.

Neomutt is a fork of mutt which retains compatibility with mutt configuration
files. One of the advantages of using neomutt over mutt is that it has support
for sending emails with a multipart/alternative content type. This means that
you can send both HTML content (with fancier formatting for email clients which
support it) and plain-text content (great for viewing in email clients like
mutt) in the same email, and simply let email clients decide which version they
prefer to display.

However, manually writing out every email in both plain-text and HTML would be
tiresome. It’s much easier to just write emails once, in a user-friendly markup
language such as markdown and let software such as pandoc convert it into HTML
and plain text.

With help from neomutt macros, this conversion can be integrated with neomutt,
so that there’s no need to exit neomutt or significantly alter your workflow
for sending emails.

  * Configuration
      + 1. Installing pandoc
      + 2. Set up a pandoc template for HTML emails
      + 3. Configure the mutt macro
      + 4. Sending your first email
  * How does it work?
      + What does the neomutt macro do?
      + Can I put inline images in these messages?
      + Why use a custom pandoc template?
      + Can I write emails in LaTeX/Haddock/[language]?
  * Video of final workflow

Configuration

This writeup assumes that you have neomutt installed and working. If not, the
following example configuration files might be useful:

  * A sample set of configuration files for neomutt with two accounts, using
    the SMTP client built into neomutt. This is my favoured approach at the
    moment, because the inbuilt SMTP support within (neo)mutt is good enough
    for my purposes, and I don’t want an MTA installed locally.
  * A more complex configuration using offlineimap to sync IMAP folders and
    mSMTP MTA. This could be helpful for later integration with powerful search
    programs, such as notmuch.
  * The Arch Linux wiki page for mutt - most configuration options are directly
    compatible between mutt and neomutt.
  * A solarized colourscheme for mutt/neomutt.

Once you’ve got neomutt installed and working, then you can proceed with the
following instructions.

1. Installing pandoc

On my system, I chose to run pandoc inside a docker container. The reason for
this is that the Arch Linux pandoc package comes with over 100 dependencies,
totally nearly a gigabyte, which would increase the number of packages
installed on my system by more than 15%.

To start using pandoc in docker, first install docker and then download the
official pandoc docker image by running docker pull pandoc/core.

For those who don’t like docker, it’s also entirely possible to run this
workflow using pandoc installed through your favourite package manager (or
compiled directly from source). In those cases, the docker commands within the
neomutt macro will need to be adjusted to call pandoc directly.

2. Set up a pandoc template for HTML emails

Add the following HTML template to ~/.mutt/templates/email.html.

    ~/.mutt/templates/email.html

1  <!DOCTYPE html>
2  <html xmlns="http://www.w3.org/1999/xhtml" lang="$lang$" xml:lang="$lang$"$if(dir)$ dir="$dir$"$endif$>
3  <head>
4    <meta charset="utf-8" />
5    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=yes" />
6    <style>
7      $styles.html()$
8    </style>
9  $for(css)$
10   <link rel="stylesheet" href="$css$" />
11 $endfor$
12   <!--[if lt IE 9]>
13     <script src="//cdnjs.cloudflare.com/ajax/libs/html5shiv/3.7.3/html5shiv-printshiv.min.js"></script>
14   <![endif]-->
15 $for(header-includes)$
16   $header-includes$
17 $endfor$
18 </head>
19 <body>
20   $body$
21   $for(include-after)$
22   $include-after$
23   $endfor$
24 </body>
25 </html>

3. Configure the mutt macro

Adding the code below to your ~/.muttrc sets up a macro which compiles markdown
to HTML and plain text when you press “m” after composing an email.

    ~/.muttrc (partial excerpt)

1 macro compose m \
2 "<enter-command>set pipe_decode<enter>\
3 <pipe-message>docker run -i -v /tmp:/tmp --rm pandoc/core -f gfm -t plain -o /tmp/msg.txt<enter>\
4 <pipe-message>docker run -i -v /tmp:/tmp -v ~/.mutt/templates/email.html:/mutt/templates/email.html --rm pandoc/core -s -f gfm --self-contained -o /tmp/msg.html --resource-path /mutt/templates/ --template email<enter>\
5 <enter-command>unset pipe_decode<enter>\
6 <attach-file>/tmp/msg.txt<enter>\
7 <attach-file>/tmp/msg.html<enter>\
8 <tag-entry><previous-entry><tag-entry><group-alternatives>" \
9 "Convert markdown to HTML5 and plaintext alternative content types"

Mutt macros can be a little intimidating at first, so there’s a line by line
description of what this macro does later in this article.

4. Sending your first email

Once you’ve typed your markdown email in your favourite text editor, you’ll
arrive at the neomutt compose screen, where you’d usually rename attachments,
edit the subject, or press ‘y’ to send the message, with the default key
bindings.

Before pressing ‘y’ to send the message:

 1. Press ‘m’ to run the macro. All being well, this will automatically create
    another inline attachment containing both the HTML and the plain text.
 2. If you’re happy with the new attachment that got generated, then delete
    your markdown file by using the arrow keys to navigate the list of
    attachments and detaching the original text (pressing Shift and D with the
    default key bindings).
 3. Press send!

It’s worth experimenting a little bit with emails to yourself, to make sure
things are actually appearing how you want.

Once you’re familiar with the process, you can even adjust your email signature
to use markdown for formatting.

How does it work?

The configuration above works well for me - but an explanation of the setup is
given below for those who may wish to customise it.

What does the neomutt macro do?

A line-by-line description of the neomutt macro is given below. The neomutt
documentation provides further detail on configuring macros.

 1. macro compose m
      + This line tells neomutt that a new macro is being bound to the “m” key
        for the email “compose” screen.
 2. <enter-command>set pipe_decode<enter>
      + This sets pipe_decode which tells neomutt to decode the email it sends
        to the next stages of the macro.
      + Decoding the email piped to later stages of the macro means that the
        next stages of the macro only receive the text content of the email,
        rather than the entire email including headers.
 3. <pipe-message>docker run -i -v /tmp:/tmp --rm pandoc/core -f gfm -t plain
    -o /tmp/msg.txt<enter>
      + This line begins with <pipe-message>, which pipes the message you
        composed to the following command.
      + The following command runs the pandoc docker image to produce a
        plaintext email (from the markdown message you wrote), which it saves
        to /tmp/msg.txt.
 4. <pipe-message>docker run -i -v /tmp:/tmp -v ~/.mutt/templates/email.html:/
    mutt/templates/email.html --rm pandoc/core -s -f gfm --self-contained -o /
    tmp/msg.html --resource-path /mutt/templates/ --template email<enter>
      + This line again pipes the message to docker/pandoc.
      + This docker/pandoc command converts the markdown message to a HTML
        message, using the template at ~/.mutt/templates/email.html.
 5. <enter-command>unset pipe_decode<enter>
      + This line unsets the change made on line 2, so other macros you may use
        afterwards are not affected.
 6. <attach-file>/tmp/msg.txt<enter>
      + Attaches the plain-text message to the email.
 7. <attach-file>/tmp/msg.html<enter>
      + Attaches the HTML message to the email.
 8. <tag-entry><previous-entry><tag-entry><group-alternatives>
      + First this line tags (selects) the current attachment (the HTML
        message).
      + This line then moves to the previous attachment, and tags that too.
      + Finally, it groups the two selected attachments together as
        alternatives, which is necessary to achieve the correct content-type on
        the email.
      + Having the correct content-type means that most clients will
        automatically select the plain-text or the HTML, depending upon what
        they are capable of displaying.
 9. "Convert markdown to HTML5 and plaintext alternative content types"
      + This line is the macro description, which is shown in the help screen
        in neomutt. The help screen is displayed by pressing “?”.

Can I put inline images in these messages?

Kind of - technically images can be embedded, but the images won’t display in
most email clients. It’s better to attach images to the email instead, and
reference them from the text.

If you really want to put inline images in (bearing in mind they won’t display
for most people), then you can copy your images to /tmp, where the docker
container will be able to access them. You can then include them in the email
using the standard markdown syntax for images (![alt text](/path/to/
image.png)), making sure to reference images using an absolute path (/tmp/
example.png).

It’s worth paying extra attention to the alt-text, because that will be very
prominent for the majority of users to whom the image will not appear: users
viewing the plain-text version of your email, and the majority of users with
clients that won’t display the image.

Putting images inline (sort of) works because the pandoc command is run with
the option --self-contained, which means that any images in the email text will
be encoded into base64 and stored within the HTML email itself. In recent
versions of pandoc, you can even adjust the size of the images by writing
markdown like ![alt text](/tmp/image.png){width=300px height=200px}. In order
to enable this feature, replace the two occurrences of -f gfm in the macro
definition with -f markdown+link_attributes. That slightly changes the markdown
syntax for the document (from GitHub Flavoured Markdown to Pandoc Markdown),
but the differences will be negligible for most.

Why use a custom pandoc template?

Using the default HTML5 template provided with pandoc generates a title element
within the email, which has the content “-“. This title element doesn’t appear
on most clients, but it does show prominently on some, such as Thunderbird.
It’s not ideal to have emails which display differently depending what software
is used to view them. Thankfully, there are a few possible approaches to
solving this:

  * setting the title of the HTML email to be the email subject. This can be
    achieved by using a bash script to extract the subject from the email, and
    then passing the subject into the pandoc command (e.g. --metadata title=
    $SUBJECT). However, the default HTML5 pandoc template will also prepend
    this title (in a H1 element) to the main email body, which would lead to it
    displaying before your email content. This would look quite unprofessional;
  * or setting the HTML title to whitespace. This can be done by adding
    --metadata title=' ' to the pandoc command that creates the HTML file. It’s
    a hacky solution, and it leads to an H1 element containing a space being
    created within the document when using the default pandoc HTML5 template,
    which could cause problems with screen readers or other programs which
    parse the HTML mail;
  * or using a custom HTML template for pandoc which does not have a title
    attribute.

The latter solution - to leave out the title element from the generated HTML -
is the solution that seems most likely to avoid differences in formatting/
display between different email clients viewing the same email. Furthermore,
leaving out the title element is actually compatible with the HTML5
specification, which states:

    The title element is a required child in most situations, but when a
    higher-level protocol provides title information, e.g. in the Subject line
    of an e-mail when HTML is used as an e-mail authoring format, the title
    element can be omitted.

The template used in this setup was created from the default pandoc HTML5
template, which can be extracted using pandoc -D html5 and then deleting the
unwanted elements - mostly those related to the title and header attributes.

Can I write emails in LaTeX/Haddock/[language]?

Any formats supported by pandoc should work. For example, to write emails in
LaTeX and have them compiled to HTML and plaintext, just change -f gfm to -f
latex in the docker commands within the macro you added to .muttrc. The pandoc
manual has a list of supported input formats.

Video of final workflow

© 2024 Tom Wemyss

## Original post 2

[Original](https://jonathanh.co.uk/blog/multipart-emails-in-neomutt/)

Multipart Emails in Neomutt

2022-05-27

It recently came to my attention that mutt now supports sending multipart
emails. I thought that this would mean that in half an hour or so I would have
html emails working. Turns out, I was wrong. What instead happened was weeks of
trial and error and reading RFCs.

I now have a system I am happy with. I write an email in markdown and mutt
(along with some surrounding scripts) will convert that markdown to html,
attach inline images and create a multipart email.

A note about HTML emails

If you do need to send HTML emails, please spare a thought for your recipient;
it is not just “weird” terminal email client users that could suffer. According
to the National Eye Institute, one in twelve men are colour blind; so please
don’t use only colour to distinguish items. Additionally, many people suffer
from vision-loss blindness, who are likely to use screen readers or braille
displays. Complex layouts or heavy use of images are going to give these people
a poor experience.

That being said, HTML emails can be used for aesthetic purposes, whilst still
providing a plain text version for those who want it. That is the method I
suggest adopting if you need html emails, and the method this blog post will
describe.

Multipart / Related emails

To begin with, we need to understand a little bit about how emails are
structured. Below is an example tree structure of a standard email.

Multipart Related
├─>Multipart Alternative
│ ├─>Plain Text Email
│ └─>HTML Email
└─>Inline Image Attachment
Non-Inline Attachment

Starting at the lowest level, we see a plain text email and an HTML email.
These are both wrapped in a multipart alternative wrapper. This tells email
clients receiving the email that they are alternative versions of the same
document. The email client will normally choose which to display based on the
mime type and user preferences.

The multipart alternative wrapper and an image attachment are then wrapped in a
multipart related wrapper. This tells the email client that the contents are
related to one another, but not different version of the same document. This is
where inline images are attached.

Finally, there is another attachment that is outside of the multipart related
wrapper. This will show up as another attachment but cannot be displayed
inline.

Neomutt Configuration

The conversion from markdown to html will be handled by an external script. It
will create files and instruct mutt to attach them.

We can start with the following:

macro compose Y "<first-entry>\
<pipe-entry>convert-multipart<enter>\
<enter-command>source /tmp/neomutt-attach-macro<enter>

We specify a macro to run when Y is pushed. First, we select the first entry.
This is in case we have attached anything manually, the first entry should be
the markdown file.

We then pipe the selected entry (the markdown file) to an external script, in
this case a bash script called convert-multipart. Finally we source a file
called /tmp/neomutt-commands. This will be populated by the script and will
allow us to group and attach files inside neomutt.

Converting to HTML

Let’s start with a simple pandoc conversion.

 #!/usr/bin/env bash
 
 commandsFile="/tmp/neomutt-commands"
 markdownFile="/tmp/neomutt-markdown"
 htmlFile="/tmp/neomutt.html"
 
 cat - > "$markdownFile"
 echo -n "push " > "$commandsFile"
 
 pandoc -f markdown -t html5 --standalone --template ~/.pandoc/templates/email.html "$markdownFile" > "$htmlFile"
 
 # Attach the html file
 echo -n "<attach-file>\"$htmlFile\"<enter>" >> "$commandsFile"
 
 # Set it as inline
 echo -n "<toggle-disposition>" >> "$commandsFile"
 
 # Tell neomutt to delete it after sending
 echo -n "<toggle-unlink>" >> "$commandsFile"
 
 # Select both the html and markdown files
 echo -n "<tag-entry><previous-entry><tag-entry>" >> "$commandsFile"
 
 # Group the selected messages as alternatives
 echo -n "<group-alternatives>" >> "$commandsFile"

The above bash script will create an html file using pandoc, and create a file
of neomutt commands. This instructs neomutt to attach the html file, set its
disposition, and group the markdown and html files into a “multipart
alternatives” group.

Neomutt’s attachment view should look something like this.

  I   1 <no description>                             [multipa/alternativ, 7bit, 0K]
- I   2 ├─>/tmp/neomutt-hostname-1000-89755-7    [text/plain, 7bit, us-ascii, 0.3K]
- I   3 └─>/tmp/neomutt.html                      [text/html, 7bit, us-ascii, 9.5K]

Inline attachments

The next part of the puzzle is inline attachments. These need to be attached
and then grouped within a multipart related group.

To reference the file from within the html email, each inline image needs a
unique cid. I use md5 sums for this. They are not cryptographically secure, but
for the purposes of generating unique strings for images in an email, they are
fine.

 grep -Eo '!\[[^]]*\]\([^)]+' "$markdownFile" | cut -d '(' -f 2 |
     grep -Ev '^(cid:|https?://)' | while read file; do
         id="cid:$(md5sum "$file" | cut -d ' ' -f 1 )"
         sed -i "s#$file#$id#g" "$markdownFile"
     done

We loop through all the images in the markdown file, and replace the paths for
cids (assuming they are not already cids or remote images).

As the markdown has changed, we need to attach the new one and detach the old.

 if [ "$(grep -Eo '!\[[^]]*\]\([^)]+' "$markdownFile" | grep '^cid:' | wc -l)" -gt 0 ]; then
     echo -n "<attach-file>\"$markdownFile\"<enter><first-entry><detach-file>" >> "$commandsFile"
 fi

To attach the images, we loop through the original file and add to the file
neomutt sources. Neomutt will be instructed to attach, set the disposition, set
the content ID and tag the image.

 grep -Eo '!\[[^]]*\]\([^)]+' "${markdownFile}.orig" | cut -d '(' -f 2 |
     grep -Ev '^(cid:|https?://)' | while read file; do
     id="$(md5sum "$file" | cut -d ' ' -f 1 )"
     echo -n "<attach-file>\"$file\"<enter>" >> "$commandsFile"
     echo -n "<toggle-disposition>" >> "$commandsFile"
     echo -n "<edit-content-id>^u\"$id\"<enter>" >> "$commandsFile"
     echo -n "<tag-entry>" >> "$commandsFile"
 done

 if [ "$(grep -Eo '!\[[^]]*\]\([^)]+' "$markdownFile" | grep '^cid:' | wc -l)" -gt 0 ]; then
     echo -n "<first-entry><tag-entry><group-related>" >> "$commandsFile"
 fi

Finally, if there were any images attached, we select the first entry (the
multipart alternative we’ve already created), tag it and mark everything tagged
as multipart related.

  I     1 <no description>                            [multipa/related, 7bit, 0K]
  I     2 ├─><no description>                      [multipa/alternativ, 7bit, 0K]
- I     3 │ ├─>/tmp/neomutt-markdown           [text/plain, 7bit, us-ascii, 0.3K]
- I     4 │ └─>/tmp/neomutt.html                [text/html, 7bit, us-ascii, 9.5K]
  I     5 └─>/tmp/2022-05-27T15-02-20Z.png              [image/png, base64, 0.5K]

At this point, the user is free to attach additional, non inline documents as
normal. This email should be good for both text based and graphical email
clients.

For the full source changes, see this commit.

My face Jonathan Hodgson

  * Advent of Code
  * Firefox
  * FZF
  * Home Assistant
  * Linux
  * Mutt
  * My New Home
  * Pentesting
  * Privacy
  * Security
  * Vim
  * Websites
  * ZSH

Help Me Out Other Stuff You Might Like Mastodon Website Source
