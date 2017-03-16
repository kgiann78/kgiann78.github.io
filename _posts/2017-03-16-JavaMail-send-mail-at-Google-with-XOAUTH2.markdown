---
layout: post
title:  "JavaMail: How to send email from Google with XOAUTH2"
date:   2017-03-16 17:30:01 +0200
categories: Java GMail
---

The original articles are here [Chariot Solutions: Sending Mail via GMail with JavaMail] and here [Java.net JavaMail OAuth2 Support] but in order to get my application to work I had to make some changes that included parts from both articles.

If you use JavaMail and/or SimpleJavaMail or the Spring MailSender to send email through a Google mail account, you get an authentication error saying Google mail blocks less secure apps.

At that point, two things are left for you to do:

1. Configure the Google account to accept connections from less secured applications [less secure app switch]
2. Configure your JavaMail sender to use OAuth for authentication

# OAuth Basics
Instead of usernames and passwords, OAuth deals with tokens.

First, you set up an application. Then you get an application ID and "secret" (a string of text), with which you can generate tokens.

Next, your application requests access to a specific GMail account (the one you plan to send mail from). Somebody with access to that account will need to visit a Web page and grant access to that application for the GMail account in question. That will generate another string of text which you can convert into tokens.

The first token is the "refresh token". This is generally long-lived, expiring after six months of disuse or after too many other refresh tokens have been generated.

The second token is the "access token". This generally lasts about an hour. This is the one you need in order to actually connect to GMail to send mail.

So then, you do all this setup, and your connection procedure looks like this:

1. Check whether you have a valid (unexpired) access token
2. If not, use your refresh token to generate a new access token
3. Connect to GMail using the access token instead of a password, and send mail
4. Profit!

# Setting up for OAuth authentication
The specific steps for the first-time setup look like this:

1. Go to https://console.developers.google.com/ and create a new "project". You will need a Google account, which may or may not be the same as the one you intend to send mail from.
2. When the project dashboard comes up for the new project, select "Enable and manage APIs"
3. Select "Credentials" on the left
4. Select "OAuth Consent Screen" on the right
5. Enter a product name in "Product name shown to users". When you need to approve access to the GMail account, you'll see a message like "Do you want to allow [Product Name] to have full control over your GMail account?" so it should be a name someone will recognize.
6. Hit "Save" and go to the "Credentials" tab
7. Hit "Create Credentials" then "OAuth Client ID"
8. Select the application type "Other" and then enter a name or description for the application (this one is not shown to users)
9. This gives you a popup with your Client ID and Client Secret. Save both of those. You will need them for all the future steps.
10. Generate a refresh key and your first access key. If you can run Python scripts, the easiest way is to run the [oauth2.py file here] (click it and then view Raw to download it). The top of the file has the parameters you need to pass when you run it, under "1. The first mode is used to generate and authorize an OAuth2 token." You'll need the GMail account you want to send mail from, and the Client ID and Client Secret generated above.
11. When you run the Python script with that syntax, it will tell you to visit a Web URL. Copy and paste that into a browser. That's where you'll get the prompt for whether you want to allow the application [Product Name] to have full control over the GMail account in question. When that is approved, it will generate another text string for you.
12. Copy that text string back into the waiting Python application. At that point it will emit the refresh token, your first access token, and the lifetime (in seconds) of the access token. Save at least the refresh token.

You should do all this only once. You will be left with a Client ID, Client Secret, and Refresh Token, all of which are strings of text. You can save the first Access Token if you like; if you do it's worth noting the current time so you can calculate when it will expire.

# Configuring JavaMail for OAuth authentication
First, you must have the JavaMail reference implementation, version 1.5.2 or later. As of this writing, this is newer than what's available in Maven or the JavaEE download package. You can get it from the JavaMail download site. The token response is formatted as JSON, so I've used Jackson to parse it in this example, though it's simple enough that virtually anything would work.

Your code should have access to your Client ID, Client Secret, and Refresh Token. A simple example (without very intelligent handling if the operation fails) would look something like this:

    String TOKEN_URL = "https://www.googleapis.com/oauth2/v4/token";
    String oauthClientId = "fixme.apps.googleusercontent.com";
    String oauthSecret = "fixme";
    String refreshToken = "fixme";
    String accessToken = "fixme";
    long tokenExpires = 1458168133864L;

    if (System.currentTimeMillis() > tokenExpires) {
        try {
            String request = "client_id=" + URLEncoder.encode(oauthClientId, "UTF-8")
                    + "&client_secret=" + URLEncoder.encode(oauthSecret, "UTF-8")
                    + "&refresh_token=" + URLEncoder.encode(refreshToken, "UTF-8")
                    + "&grant_type=refresh_token";
            HttpURLConnection conn = (HttpURLConnection) new URL(TOKEN_URL).openConnection();
            conn.setDoOutput(true);
            conn.setRequestMethod("POST");
            PrintWriter out = new PrintWriter(conn.getOutputStream());
            out.print(request); // note: println causes error
            out.flush();
            out.close();
            conn.connect();
            try {
                HashMap<String, Object> result;
                result = new ObjectMapper().readValue(conn.getInputStream(), new TypeReference<HashMap<String, Object>>() {
                });
                accessToken = (String) result.get("access_token");
                tokenExpires = System.currentTimeMillis() + (((Number) result.get("expires_in")).intValue() * 1000);
            } catch (IOException e) {
                String line;
                BufferedReader in = new BufferedReader(new InputStreamReader(conn.getErrorStream()));
                while ((line = in.readLine()) != null) {
                    System.out.println(line);
                }
                System.out.flush();
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

The script above will refresh the initial token if necessary. Then you can use the accessToken to your code:

1. Create the properties:
    ##### For JavaMail 1.5.5 and later
    Starting with JavaMail 1.5.5, support for [OAuth2 authentication] is built-in and no longer requires SASL (although the SASL OAuth2 support continues to work).

    Since OAuth2 uses an "access token" instead of a password, you'll want to configure JavaMail to use only the XOAUTH2 mechanism. The access token is passed as the password, which obviously won't work with any of the other authentication mechanisms. For example, to access Gmail:
    
    For imap:

        Properties props = new Properties();
        props.put("mail.imap.ssl.enable", "true"); // required for Gmail
        props.put("mail.imap.auth.mechanisms", "XOAUTH2");

    For smtp:

        Properties props = new Properties();
        props.put("mail.smtp.ssl.enable", "true"); // required for Gmail
        props.put("mail.smtp.auth.mechanisms", "XOAUTH2");
    
    ##### For JavaMail 1.5.2 to 1.5.4
    For JavaMail 1.5.2 to 1.5.4, support for OAuth2 authentication via the SASL XOAUTH2 mechanism is included.

    The SASL XOAUTH2 provider will be added to the Java security configuration when SASL support is first used. The application must have the permission SecurityPermission("insertProvider.JavaMail-OAuth2").

    Since OAuth2 uses an "access token" instead of a password, you'll want to configure JavaMail to use only the SASL XOAUTH2 mechanism. The access token is passed as the password, which obviously won't work with any of the other authentication mechanisms. For example, to access Gmail:
    
    For imap:
    
        Properties props = new Properties();
        props.put("mail.imap.ssl.enable", "true"); // required for Gmail
        props.put("mail.imap.sasl.enable", "true");
        props.put("mail.imap.sasl.mechanisms", "XOAUTH2");
        props.put("mail.imap.auth.login.disable", "true");
        props.put("mail.imap.auth.plain.disable", "true");

    For smtp:
    
        Properties props = new Properties();
        props.put("mail.smtp.ssl.enable", "true"); // required for Gmail
        props.put("mail.smtp.sasl.enable", "true");
        props.put("mail.smtp.sasl.mechanisms", "XOAUTH2");
        props.put("mail.smtp.auth.login.disable", "true");
        props.put("mail.smtp.auth.plain.disable", "true");

2. After the properties file is set up, then we send mail like normal:

        Message msg = new MimeMessage(session);
        msg.setFrom(new InternetAddress(mailUsername));
        msg.addRecipient(Message.RecipientType.TO, new InternetAddress(mailUsername));

        msg.setSubject("JavaMail OAuth2 test");
        msg.setSentDate(new Date());
        msg.setText("Hello, world with OAuth2!\n");
        msg.saveChanges();

    For imap:
    
        Session session = Session.getInstance(props);
        Store store = session.getStore("imap");
        store.connect("imap.gmail.com", username, accessToken);

    For smpt:
        
        Session session = Session.getInstance(props);
        Transport transport = session.getTransport("smpt");
        transport.connect("smtp.gmail.com", username, accessToken);
        transport.send(msg, msg.getAllRecipients());


[Chariot Solutions: Sending Mail via GMail with JavaMail]:<http://chariotsolutions.com/blog/post/sending-mail-via-gmail-javamail/>
[Java.net JavaMail OAuth2 Support]:<https://java.net/projects/javamail/pages/OAuth2>
[less secure app switch]:<https://www.google.com/settings/security/lesssecureapps>
[oauth2.py file here]:<https://github.com/google/gmail-oauth2-tools/tree/master/python>
[OAuth2 authentication]:<https://developers.google.com/gmail/xoauth2_protocol>
