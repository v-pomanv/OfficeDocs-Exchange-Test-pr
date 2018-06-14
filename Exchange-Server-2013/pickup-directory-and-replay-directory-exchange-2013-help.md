﻿---
title: 'Pickup directory and Replay directory: Exchange 2013 Help'
TOCTitle: Pickup directory and Replay directory
ms:assetid: ae191700-953f-411c-906f-dc90feec3d5a
ms:mtpsurl: https://technet.microsoft.com/en-us/library/Bb124230(v=EXCHG.150)
ms:contentKeyID: 49382861
ms.date: 12/09/2016
mtps_version: v=EXCHG.150
---

# Pickup directory and Replay directory

 

_**Applies to:** Exchange Server 2013_


By default, the Pickup and Replay directories exist on every Microsoft Exchange Server 2013 Mailbox server or Edge Transport server. Correctly formatted email message files that you copy to the Pickup or Replay directories are submitted for delivery. The Pickup directory is used by administrators for mail flow testing, or by applications that must create and submit their own messages. The Replay directory receives messages from foreign gateway servers and can also be used to resubmit messages that administrators export from the queues of Exchange servers.

**Contents**

Anatomy of an email message file

How the Pickup and Replay directories process messages

Pickup directory message file requirements

Pickup directory message header modifications

Replay directory message file requirements

Replay directory message header modifications

Failures in Pickup and Replay directory message processing

Security considerations for the Pickup and Replay directories

Permissions for the Pickup and Replay directories

## Anatomy of an email message file

A standard SMTP email message consists of a *message envelope* and message content. The message envelope contains information required for transmitting and delivering the message. The message content contains message header fields (collectively called the *message header*) and the message body. The message envelope is described in RFC 2821, and the message header is described in RFC 2822.

When a sender composes an email message and submits it for delivery, the message contains the basic information required to comply with SMTP standards, such as a sender, a recipient, the date and time that the message was composed, an optional subject line, and an optional message body. This information is contained in the message itself and, by definition, is contained in the message header.

The sender's messaging server generates a message envelope for the message by using the sender and recipient information found in the message header and transmits the message to the Internet for delivery to the recipient's messaging server. Recipients never see the message envelope, because it's generated by the message transmission process and isn't actually part of the message.

Each server involved in the transmission of the message may insert message header fields related to the server's role in delivering the message or other application-specific message header fields into the message header. When the recipient opens the message by using an email client, the email client displays some of the more relevant information from the message header, such as the sender, the recipients, and the subject together with the message body.

Return to top

## How the Pickup and Replay directories process messages

In Exchange 2013, the default location of the Pickup directory is `%ExchangeInstallPath%TransportRoles\Pickup`. The default location of the Replay directory is `%ExchangeInstallPath%TransportRoles\Replay`. A correctly formatted .eml message file copied to the Pickup or Replay directory is processed for submission in the following steps:

1.  The Pickup and Replay directories are checked for new message files every five seconds. You can't modify this polling interval. You can adjust the rate of message file processing by using the *PickupDirectoryMaxMessagesPerMinute* parameter on the **Set-TransportService** cmdlet. This parameter affects the Pickup directory and the Replay directory. The default value is 100 messages per minute. Files that can't be opened are left in the Pickup directory and are reevaluated at the next poll.

2.  Limits put on message files in the Pickup directory, such as the maximum header size and the maximum number of recipients, are checked. By default, the maximum header size is 64 kilobytes (KB), and the maximum number of recipients is 100. You change these limits by using the **Set-TransportService** cmdlet. These settings affect the Pickup directory only.

3.  The file is renamed from *\<filename\>*.eml to *\<filename\>*.tmp. If the *\<filename\>*.tmp file already exists, the file is renamed as *\<filename\>\<datetime\>*.tmp. If the file renaming fails, an event log error is generated, and the pickup process proceeds to the next file.

4.  After the .tmp file is successfully converted into an email message, a **delete on close** command is issued to the .tmp file. The .tmp file appears to remain in the Pickup directory, but the file can't be opened.

5.  After the message is successfully queued for delivery, a **close** command is issued, and the .tmp file is deleted from the Pickup directory. If the deletion fails, an event log error is generated. If the Microsoft Exchange Transport service is restarted when there are .tmp files in the Pickup directory, all .tmp files are renamed as .eml files and are reprocessed. This could lead to duplicate message transmission.

Return to top

## Pickup directory message file requirements

A message file copied to the Pickup directory must meet the following requirements for successful delivery:

  - The message file must be a text file that complies with the basic SMTP message format. MIME message header fields and content are supported.

  - The message file must have an .eml file name extension.

  - At least one email address must exist in the `Sender` or `From` message header fields in the message header. If a single email address exists in both the `Sender` and `From` fields, the email address in the `From` field is used as the originator of the message in the message envelope.

  - Only one email address can exist in the `Sender` field. Multiple email addresses aren't allowed. The `Sender` field is optional if only one email address exists in the `From` field.

  - Multiple email addresses are allowed in the `From` field, but a single email address must also exist in the `Sender` field. The address in the `Sender` field is then used as the originator of the message in the message envelope.

  - At least one email address must exist in the `To`, `Cc`, or `Bcc` fields.

  - A blank line must exist between the message header and the message body.

This example shows a plain text message that uses acceptable formatting for the Pickup directory.

    To: mary@contoso.com
    From: bob@fabrikam.com
    Subject: Message subject
    
    This is the body of the message.

MIME content is also supported in Pickup directory message files. MIME defines a broad range of message content that includes languages that can't be represented in 7-bit ASCII text, HTML, and other multimedia content. A complete description of MIME and its requirements is beyond the scope of this topic. This example shows a simple MIME message that uses acceptable formatting for the Pickup directory.

    To: mary@contoso.com
    From: bob@fabrikam.com
    Subject: Message subject
    MIME-Version: 1.0
    Content-Type: text/html; charset="iso-8859-1"
    Content-Transfer-Encoding: 7bit
    
    <HTML><BODY>
    <TABLE>
    <TR><TD>cell 1</TD><TD>cell 2</TD></TR>
    <TR><TD>cell 3</TD><TD>cell 4</TD></TR>
    </TABLE>

    </BODY></HTML>

Return to top

## Pickup directory message header modifications

The Pickup directory removes any of the following message header fields from the message header:

  - `Received`

  - `Resent-*`

  - `Bcc`
    

    > [!NOTE]
    > Any email addresses found in the optional <CODE>Bcc</CODE> message header fields in the message header are correctly processed. After the <CODE>Bcc</CODE> recipients are promoted to invisible message envelope recipients, they are removed from the message header to protect their identity. If a message contains only <CODE>Bcc</CODE> recipients, the value of <STRONG>Undisclosed Recipients</STRONG> is added to the <CODE>To</CODE> field in the message header.



The Pickup directory adds its own `Received` header field to a message as part of the message submission process. The `Received` header field is applied in the following format.

    Received: from localhost by Pickup with Microsoft SMTP Server id <ExchangeServerVersion><datetime>

The Pickup directory modifies the following message header fields if they're missing or malformed:

  - **Message-Id**   If the `Message-Id` field is missing or empty, the Pickup directory adds a Message-Id field by using the format *\<GUID\>*@*\<defaultdomain\>*.

  - **Date**   If the `Date` field is missing or malformed, the Pickup directory adds the date and time of message processing by the Pickup directory.

Return to top

## Replay directory message file requirements

The Replay directory is used to resubmit exported Exchange messages and to receive messages from foreign gateway servers. These messages are already formatted for the Replay directory. There is little or no need for administrators or applications to compose and submit new message files by using the Replay directory. The Pickup directory should be used to create and submit new message files.

The Replay directory messages make extensive use of *X-Headers*. X-Headers are user-defined, unofficial message header fields that exist in the message header. X-Headers aren't specifically mentioned in RFC 2822, but the use of an undefined message header field starting with "X-" has become an accepted way to add unofficial message header fields to a message. The Exchange-specific X-Headers used in the message files in the Replay directory can actually set delivery information that normally exists in the message envelope. This feature is required to preserve original message information when you use the Replay directory to process exported messages from another Exchange server.

A message file copied to the Replay directory must meet the following requirements for successful delivery:

  - The message file must be a text file that complies with the basic SMTP message format. MIME message header fields and content are supported.

  - The message file must have an .eml file name extension.

  - X-Headers must occur before all regular header fields.

  - A blank line must exist between the header fields and the message body.

The X-Headers described in the following list are required by messages in the Replay directory:

  - **X-Sender**   This X-Header replaces the `From` message header field requirement in a typical SMTP message. One `X-Sender` field that contains one email address must exist. The Replay directory ignores the `From` message header field if it's present, although the recipient's email client displays the value of the `From` message header field as the sender of the message. Other parameters usually exist in the `X-Sender` field, as shown in the following example.
    
        X-Sender: <bob@fabrikam.com> BODY=7bit RET=HDRS ENVID=12345ABCD auth=<someAuth>
    

    > [!NOTE]
    > These parameters are message envelope values that are ordinarily generated by the sending server. You may see parameters similar to this in exported message files.<BR><CODE>RET</CODE> specifies whether the whole message or only the headers should be returned to the sender if the message can't be delivered. <CODE>RET</CODE> can have a value of <CODE>HDRS</CODE> or <CODE>FULL</CODE>.<CODE> ENVID</CODE> is a message envelope identifier. <CODE>BODY</CODE> specifies the text encoding of the message. <CODE>auth</CODE> specifies an authentication mechanism to the messaging server as described in RFC&nbsp;2554.



  - **X-Receiver**   This X-Header replaces the `To` message header field requirement in a typical SMTP message. At least one `X-Receiver` field that contains one email address must exist. Multiple `X-Receiver` fields are allowed for multiple recipients. The Replay directory ignores the `To` message header fields if they're present, although the recipient's email client displays the values of the `To` message header fields as the recipients of the message. Other optional parameters may exist in the `X-Receiver` fields, as shown in the following example.
    
        X-Receiver: <mary@contoso.com> NOTIFY=NEVER ORcpt=mary@contoso.com
    

    > [!NOTE]
    > These parameters are message envelope values that are ordinarily generated by the sending server. You may see parameters similar to this in exported message files. These parameters are related to delivery status notification (DSN) messages as described in RFC&nbsp;1891.<BR><CODE>NOTIFY</CODE> can have a value of <CODE>NEVER</CODE>, <CODE>DELAY</CODE>, or <CODE>FAILURE</CODE>. <CODE>ORcpt</CODE> preserves the original recipient of the message.



The X-Headers described in the following list are optional for message files in the Replay directory:

  - **X-CreatedBy**   Used for header firewall functionality. If this X-Header exists, it must not be blank. If the `X-CreatedBy` field doesn't exist, it's added with a value of **Unspecified**. Typically, the value of this field is **MSExchange15**, but it also may contain the non-SMTP address space type set on a Send connector, such as **Notes**.

  - **X-EndOfInjectedXHeaders**   Size in bytes of all the X-Headers present. This X-Header may be used as a marker to indicate the last X-Header before the regular message header fields start.

  - **X-ExtendedMessageProps**   Extended message properties for the message.

  - **X-HeloDomain**   HELO/EHLO domain string presented during the initial SMTP protocol conversation.

  - **X-Source**   Used by Queue Viewer under the **MessageSourceName** column. If the value of this X-Header isn't specified, the value of **Replay** is used. Other possible values for this X-Header are **Smtp Receive Connector** and **Smtp Send Connector**.

  - **X-SourceIPAddress**   IP address of the sending server. This field is `0.0.0.0` if no IP address is specified.

This example shows a plain text message that uses acceptable formatting for the Replay directory.

    X-Receiver: <mary@contoso.com> NOTIFY=NEVER ORcpt=mary@contoso.com
    X-Sender: <bob@fabrikam.com> BODY=7bit ENVID=12345AB auth=<someAuth>
    Subject: Optional message subject
    
    This is the body of the message.

MIME content is also supported in Replay directory message files. MIME defines a broad range of message content that includes languages that can't be represented in 7-bit ASCII text, HTML, and other multimedia content. A complete description of MIME and its requirements is beyond the scope of this topic. This example shows a simple MIME message that uses acceptable formatting for the Replay directory.

    X-Receiver: <mary@contoso.com> NOTIFY=NEVER ORcpt=mary@contoso.com
    X-Sender: <bob@fabrikam.com> BODY=7bit ENVID=12345ABCD auth=<someAuth>
    To: mary@contoso.com
    From: bob@fabrikam.com
    Subject: Optional message subject
    MIME-Version: 1.0
    Content-Type: text/html; charset="iso-8859-1"
    Content-Transfer-Encoding: 7bit
    
    <HTML><BODY>
    <TABLE>
    <TR><TD>cell 1</TD><TD>cell 2</TD></TR>
    <TR><TD>cell 3</TD><TD>cell 4</TD></TR>
    </TABLE>

    </BODY></HTML>

Return to top

## Replay directory message header modifications

The Replay directory deletes the `Bcc` message header field from the message file.

The Replay directory adds its own `Received` message header field to a message as part of the message submission process. The Received message header field is applied in the following format.

    Received: from <ReceivingServerName> by Replay with <ExchangeServerVersion><DateTime>

The Replay directory modifies the following message header fields in the message header:

  - **Message-ID**   If this message header field is missing or empty, the Replay directory adds a Message-ID message header field by using the format *\<GUID\>*@*\<defaultdomain\>*.

  - **Date**   If this message header field is missing or malformed, the Replay directory adds the Date message header field using the date and time of message processing by the Replay directory.

Return to top

## Failures in Pickup and Replay directory message processing

A message file copied to the Pickup or Replay directories may not be successfully queued for delivery. The following categories of message submission failure can occur:

  - **Delivery failures**   A correctly formatted message file together with a valid sender that can't be successfully submitted for delivery generates a non-delivery report (NDR). Malformed content or Pickup directory message restriction violations could also cause an NDR. When an NDR is generated during message processing, the original message file is attached to the NDR message, and the message file is deleted from the Pickup directory or the Replay directory.
    

    > [!NOTE]
    > A correctly formatted message submitted into the transport pipeline may later experience a delivery failure and be returned to the sender with an NDR. This kind of failure may be caused by transmission issues unrelated to the Pickup or Replay directories, such as messaging server failures or routing failures along the delivery path of the message.



  - **Badmail**   A message classified as *badmail* has serious problems that prevent the Pickup or Replay directories from submitting the message for delivery. The other condition that causes badmail is when the message is formatted correctly, but the recipients aren't valid, and an NDR message can't be sent to the sender because the sender isn't valid.
    
    Message files determined to be badmail are left in the Pickup or Replay directories and are renamed from *\<filename\>*.eml to *\<filename\>*.bad. If the *\<filename\>*.bad file already exists, the file is renamed to *\<filename\>\<datetime\>*.bad. If badmail exists in the Pickup or Replay directories, an event log error is generated, but the same badmail messages don't generate repeated event log errors.
    

    > [!NOTE]
    > Always compose and save message files in a different location before you copy them into the Pickup directory for delivery. The Pickup directory polls for new messages every five&nbsp;seconds. Therefore, if you try to compose and save the message files in the Pickup directory itself, the Pickup directory may try to process the message files before you finish composing them.



Return to top

## Security considerations for the Pickup and Replay directories

The following list describes security concerns that are common to the Pickup directory and the Replay directory:

  - Any security checks configured on a Receive connector, such as anti-spam, anti-malware, sender filtering, or recipient filtering actions, aren't performed on messages submitted through the Pickup directory or the Replay directory.

  - A compromised Pickup directory or Replay directory can act as an open relay. This enables messages to be resubmitted or *relayed* by using a different server to mask the true source of the messages.

The following list describes additional security concerns that apply to the Replay directory:

  - The X-Headers used by the Replay directory allow for the manual creation of the message envelope. The information in the `X-Sender` and `X-Receiver` fields can be completely different from the `To` or `From` message header fields displayed by email clients. Such an impersonation of a sender and a domain is frequently called *spoofing*. A *spoofed mail* is an email message that has a sending address that was modified to appear as if it originates from a sender other than the actual sender of the message.

  - If the `X-CreatedBy` field has the value of **MSExchange15**, the destination is considered trustworthy, and header firewall isn't applied. *Header firewall* is a way for Exchange to preserve X-Headers in messages transmitted between trusted Exchange servers or to remove potentially revealing X-Headers from messages transmitted to untrusted destinations outside the Exchange organization. These X-Headers can be used to share Exchange information such as spam confidence level (SCL), message signing, or encryption between authorized Exchange servers. Revealing this information to unauthorized sources could pose a potential security risk. For more information about header firewall, see [Understanding Header Firewall](https://go.microsoft.com/fwlink/?linkid=268394).

Tighter security should be applied to the Replay directory because of the additional security risks associated with the Replay directory. Users or applications that must generate and submit messages can be granted access to the Pickup directory, but they shouldn't require access to the Replay directory.

Both the Pickup directory and the Replay directory are enabled by default on all Mailbox servers and Edge Transport servers. If the Pickup directory or the Replay directory isn't required on a specific Mailbox server or Edge Transport server in your organization, you can disable the Pickup directory or the Replay directory on that server by setting the Pickup directory path or Replay directory path to the value `$null`. For more information, see [Configure the Pickup directory and the Replay directory](configure-the-pickup-directory-and-the-replay-directory-exchange-2013-help.md).

Return to top

## Permissions for the Pickup and Replay directories

The following permissions are required on the Pickup and Replay directories:

  - Administrator: Full Control

  - System: Full Control

  - Network Service: Read, Write, and Delete Subfolders and Files

By default, the Microsoft Exchange Transport service uses the security credentials of the Network Service user account to manage the location and permissions of the Pickup and Replay directories. The Network Service account requires these permissions on the Pickup directory so that .eml files can be opened, renamed to .tmp and deleted, or renamed to .bad if the message is classified as badmail.

You can move the location of these directories by using the *PickupDirectoryPath* and *ReplayDirectoryPath* parameters on the **Set-TransportService** cmdlet. Successfully changing the location of the Pickup directory depends on the rights granted to the Network Service account at the new directory locations, and whether the new directories already exist. If the directory doesn't exist, and the Network Service account has the rights required to create folders and apply permissions at the new location, the directory is created, and the correct permissions are applied to it. If the new directory already exists, the existing folder permissions aren't checked. Whenever you move the directory locations by using the *PickupDirectoryPath* or *ReplayDirectoryPath* parameter with the **Set-TransportService** cmdlet, always verify that the new directory exists and that the new directory has the correct permissions applied to it.

Return to top
