# Final Project Assignment 2: Explore One More! (FP2) 
DUE March 30, 2015 Monday (2015-03-30)


### My Library: net/imap

Name: Yong Cho


##### IMAP: Reading Mail

I chose to play with the IMAP library. IMAP library provides procedures to connect to an email server and retrieve email contents. [**Racket IMAP document**][racket-imap] lists all the procedures availabe in the library and instructions to use them.

In order to connect to an email server, you use `imap-connect*` procedure. Of course, you need a login credentials as well as the server's internet address. Since you are sending personal information over the network, it is necessary to establish a secure connection using SSL. Racket has an openssl library for this. Following code makes a connection to the server and logs you in.

```
(require openssl)
(require net/imap)

(define SSL-PORT 993)
(define SERVER "imap.googlemail.com")
(define USERNAME "my-email-id@gmail.com")
(define PASSWORD "my-email-password")
(define MAILBOX "inbox")

(define ctx (ssl-secure-client-context))
(define-values (in out) (ssl-connect SERVER SSL-PORT ctx))

(define-values (imap-gmail total-msgs recent-msgs) 
  (imap-connect* in out USERNAME PASSWORD MAILBOX))
  
> imap-gmail
#<imap>
```

Once you are logged in, you can use `imap-messages` procedure to check how many messages are in your mailbox.

```
(define number-of-messages (imap-messages imap-gmail))
> number-of-messages
29

```


To download email messages, you can call `imap-get-messages` procedure. This procedures consumes a list of email indexes you want to download and a set of symbols indicating which fields of the message to download -- 'uid, 'header, 'body, 'flags. Here is an exmaple.

```
(define msg-list (imap-get-messages imap-gmail (list 1 3 5) '(header body)))
> msg-list
'((#"MIME-Version: 1.0\r\nReceived: by 10.64.232.6; Sun, 17 Mar 2013 17:37:54 -0700 (PDT)\r\nDate: Sun, 17 Mar 2013 17:37:54 -0700\r\nMessage-ID: <CAPseogrKLn5HWH_YztimwKWGrofQE67WsJvSnVWJ5xFW1BeF8A@mail.gmail.com>\r\nSubject: Get Gmail on your mobile phone\r\nFrom: Gmail Team <mail-noreply@google.com>\r\nTo: Yong Cho <my-email-id@gmail.com>\r\nContent-Type: multipart/alternative; boundary=bcaec53f89b72165b904d82833d3\r\n\r\n"
#"--bcaec53f89b72165b904d82833d3\r\nContent-Type: text/plain; charset=ISO-8859-1\r\nContent-Transfer-Encoding: quoted-printable\r\n\r\n [image: Access Gmail on your mobile 
... (

```

You probably want to use a regex or something to parse what you want out of the raw contents. Also, you need to use `bytes->string` procedure if you want to convert the downloaded byte string into a regular string.

```
(define parsed-subject
    (regexp-match #rx"\r\nSubject: (.*)\r\nFrom:" raw-header))
(define subject (bytes->string/utf-8 (cadr parsed-subject)))
> subject
Get Gmail on your mobile phone
```

Here is a screenshot of a simple email client I wrote using this library.

![email-client.png](https://raw.githubusercontent.com/YongCho/FP2/master/image/email-client.png)

And here is the code for it. Feel free to modify and use it.

```
#lang racket

(require racket/gui/base
         openssl
         net/imap)
;; (require browser)
;; (require browser/htmltext)

;; login information
(define SSL-PORT 993)
(define SERVER "imap.googlemail.com")
(define USERNAME "my-email-id@gmail.com")
(define PASSWORD "my-password")
(define MAILBOX "inbox")

;; GUI size
(define MIN-WIDTH 620)
(define SUBJECT-COLUMN-WIDTH 400)
(define FROM-COLUMN-WIDTH 100)
(define DATE-COLUMN-WIDTH 100)

(define SUBJECT-COLUMN 0)
(define FROM-COLUMN 1)
(define DATE-COLUMN 2)
(define columns-list (list "Subject" "From" "Date"))

(define make-message list)
(define message-subject car)
(define message-from cadr)
(define message-date caddr)
(define message-body cadddr)

(define (lmap proc lists)
  (if (null? (car lists))
      '()
      (cons (foldr proc '() (map car lists))
            (lmap proc (map cdr lists)))))

(define (zip list-of-lists)
  (lmap cons list-of-lists))

(define main-window (new frame%
                         (label "Simple E-mail Client v0.1")
                         (height 600)))

;; Top panel has the list of emails the user can select.
;; Bottom panel shows the contents of the selected email.
(define v-panel
  (new vertical-panel%
       (parent main-window)))

(define top-panel
  (new panel%
       (parent v-panel)))

(define message-list-box
  (new list-box%
       (parent top-panel)
       (label #f)
       (style '(single column-headers))
       (min-width MIN-WIDTH)
       (columns columns-list)
       (choices (list ))  ; list will be filled once the emails are downloaded.
       (callback 
        (lambda (c e)     ; called when the user selects an email
          (define selection (car (send message-list-box get-selections)))
          (define mail-body (send message-list-box get-data selection))
          (send contents-text erase)
          (send contents-text insert mail-body)
          (send contents-canvas scroll-to 0 0 0 0 #t 'start)))))
 
(send message-list-box set-column-width SUBJECT-COLUMN SUBJECT-COLUMN-WIDTH 100 1920)
(send message-list-box set-column-width FROM-COLUMN FROM-COLUMN-WIDTH 30 300)
(send message-list-box set-column-width DATE-COLUMN DATE-COLUMN-WIDTH 30 200)

(define bottom-panel
  (new panel%
       (parent v-panel)
       (style (list 'border))))

(define contents-canvas
  (new editor-canvas% (parent bottom-panel)))
(define contents-text (new text%))
(send contents-text auto-wrap #t)
(send contents-canvas set-editor contents-text)

(send main-window show #t)

;; Send the login information to the server.
(define ctx (ssl-secure-client-context))
(define-values (in out) (ssl-connect SERVER SSL-PORT ctx))
(define-values (imap-gmail total-msgs recent-msgs) 
  (imap-connect* in out USERNAME PASSWORD MAILBOX))

;; Downloads ith message in the mailbox (counting from 1).
(define (get-msg i)
  (define msg-list (imap-get-messages imap-gmail (list i) '(header body)))
  (define raw-header (caar msg-list))
  (define raw-body (cadar msg-list))
  (define parsed-subject
    (regexp-match #rx"\r\nSubject: (.*)\r\nFrom:" raw-header))
  (define parsed-from
    (regexp-match #rx"\r\nFrom: (.*) <.*>\r\nTo: " raw-header))
  (define parsed-date
    (regexp-match #rx"\r\nDate: [a-zA-Z]*, ([0-9]* [a-zA-Z]* [0-9]*) " raw-header))
  (cond ((not (and parsed-subject parsed-from parsed-date)) #f)
        (else         
         (define subject (bytes->string/utf-8 (cadr parsed-subject)))
         (define from (bytes->string/utf-8 (cadr parsed-from)))
         (define date (bytes->string/utf-8 (cadr parsed-date)))
         (define body (bytes->string/utf-8 raw-body))
         (make-message subject from date body))))

;; Downloads ith to end'th messages.
;; Returns them as a list of lists.
(define (get-msgs i end)
  (cond ((> i end) '())
        ((not (get-msg i)) (get-msgs (+ 1 i) end))
        (else (cons (get-msg i)
                    (get-msgs (+ 1 i) end)))))

;; Adds a new message into the email list GUI panel.
(define (add-new-message msg)
  (send message-list-box append (message-subject msg))
  (define number-of-messages (send message-list-box get-number))
  (define last (- number-of-messages 1))
  (send message-list-box set-string last (message-from msg) FROM-COLUMN)
  (send message-list-box set-string last (message-date msg) DATE-COLUMN)
  (send message-list-box set-data last (message-body msg)))

;; Adds ith to end'th messages to the email list GUI.
(define (add-messages i end)
  (cond ((> i end) (void))
        ((not (get-msg i)) (add-messages (+ 1 i) end))
        (else (add-new-message (get-msg i))
              (add-messages (+ 1 i) end))))

;; Go ahead and download all the messages and show them on the GUI.
(define number-of-messages (imap-messages imap-gmail))
(if (> number-of-messages 0)
    (add-messages 1 (imap-messages imap-gmail))
    (void))



```

##### Summary
Racket's IMAP library works as advertised.


##### Other thoughts
Most email nowadays contain a boat load of html tags and hyperlinked images. It would have been nice if Racket had a way to render an html contents. I don't think Racket's browser library has an option for this. I could be wrong. It's something to check for.


<!-- Links -->
[racket-imap]: http://docs.racket-lang.org/net/imap.html