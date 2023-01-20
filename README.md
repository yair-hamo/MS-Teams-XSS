# XSS and other vulnerabilities on Microsoft Teams.

During one of my researches, I found some interesting vulnerabilities on MSTeams. Here I will intreduce some of them.  
I was able to run code in another user's application, even when the sender is not part of the recipient's organization.  
**Clarification**: All the vulnerabilities we present below have already been fixed by Microsoft.  
<p></p><img src="imgs/xss_demo.gif">

## Intro to Teams messaging API

Teams holds subdomains under *.msg.teams.microsoft.com
- **Request**:
<img src="imgs/simple_msg_req.png"></img>   
- **Appearance on app**
<img src="imgs/simple_msg_res.png"></img>
As we can see from the above (I removed any unnecessary header) for sending a message request we need:
1. **Authorization token** - token we get when we perform login.
2. **Message thread ID** - at the URL path we have the conversation thread ID.
   
The body of the message structures on HTML format under **content** section.  
We can use whatever HTML tag we want, but Teams' server sanitizes the html and leaves only desired tags and strips them, and of course, removes any JavaScript usage.
Over the years many escapes from the html sanitising has been found on Teams application - [XSS + RCE example](https://github.com/oskarsve/ms-teams-rce/blob/main/README.md).  
I founded an interesting vulnerability which I will describe below.

## Abusing Code-snippet sharing feature

One of the nicest Teams feature is code-snippet sharing.  
The basic idea of this feature is to enable code sharing over chat, with code sytax highlighting for many languages.  
I did some observations on this feature's API flow and found something interesting.  
First let's break the code sharing steps.

### 1. Code-snippet sharing - getting share-ID

In order to share a code snippet, the client needs first to send the other person's client ID(s) with the desired permissions (read / write) for getting **share-ID token** as we can see below:  

<img src="imgs/getting_share_id.png"></img>
   
### 2. Code-snippet sharing - uploading the code to the cloud

After getting the share-id, we can now upload the code snippet to the cloud as we can see below:  

<img src="imgs/code_share.png"></img>
    
As you can see, there is a very interesting delimiter withing the shared message body data: ```~~~~###@@TACOCAT@@###~~~~``` which seperates the clear code, and the way of presenting it over HTML.  
For me that seems as an intersting point to abuse, so I removed the delimiter and injected my own html content.  
To my supprise, I found out that Teams' backend filters my html totaly different than on normal message, so I started trying to inject some xss payloads.  

#### CSP bypassing
The main problem with XSS on the Teams app is the [Content Security Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP) which doesn't seems to have a easy hole in it as we can see using [Content Security Policy (CSP) Generator](https://chrome.google.com/webstore/detail/content-security-policy-c/ahlnecfloencbkpfnpljbojmjkfgnmdc) Chrome extension and evaluting it at [csp-evaluator](https://csp-evaluator.withgoogle.com/):
  
<img src="imgs/scp_list.png"></img>

### 3. Code-snippet sharing - finding XSS on the upload process

As I explained before, using the delimiter bypass and modifying the html code in the code upload message, only partial list of html tags bypasess the html sanitizer. In this case, in order to bypass CSP, I thought about poping a new window. reminder - we are talking about Teams native application which based on [Electron](https://www.electronjs.org/).  
If I will manage to pop a new window using **target** attribute, I will be able to navigate to any webpage I want, and maybe to [escape Electron's sandboxing](https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/xss-to-rce-electron-desktop-apps) which still seems to be possible task these days.  
On PortSwigger's [cheat sheet](https://portswigger.net/web-security/cross-site-scripting/cheat-sheet#set-windowname-via-usemap-attribute-in-a-img-tag) I found an interesting tag that can be useful here, ```<map> and <area>```. Those tags with modified target _self attribute bypasses the html sanitizer and opens whatever webpage I want.  

### 3. Code-snippet sharing - sending the shared code within a message
The last step, sharing the code-snippet within a message.
Here we will need to attach the share-id.  

<img src="imgs/sending_shared_code.png"></img>
  
And the final solution:  

<img src="imgs/shared_code_res.png"></img>

### XSS demo
As you can see, the payload was sent from extenal user outside any company, which mean that at the time, every company was vulnerable to such attack.

### Trying to achive RCE and a sad ending

I worked on this exploit for couple of days, and was trying to understand teams' Electron backend messaging protocol and vulnerabilities. I reverse engineered their source code from the ASAR file and tried to find an interesting holes.  
I managed to do many things such as changing the window dimentions, downloading files, and more. But still not to perform a real RCE.  
Then came the happy ending, when Microsoft fixed the issue on their new version of Teams.  
