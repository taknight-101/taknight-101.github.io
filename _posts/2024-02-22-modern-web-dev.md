---
layout: post
title: "Modern Web Development with React in ServiceNow"
permalink: "modern-web-dev"
categories: [Platform Engineering, ServiceNow]
tags: ["r&d" , "react" , "modern-web-development"  ,"web_bundlers" , "scripted_rest_api"]  # TAG names should always be lowercase
image: /images/modern-web-dev/react/react.jpg


---




## _In this series of blogs, we will show, how it's possible to employ modern web development techniques in ServiceNow_ 

<u>Typically web sites falls into 2 main categories </u>


1. [MPA Multi Page Applications](https://www.ohmycrawl.com/faq/what-is-a-multi-page-application/): 
sometimes called SSP or Server Sent Pages

In this type, the browser reaches out to the server upon each user request to fetch the page content , introducing network latency and poor performance 


2. [SPA - Single Page Applications](https://www.bloomreach.com/en/blog/2018/what-is-a-single-page-application): 

In which you interact with the serve via APIs to mainly get the data powering your application and handling user interface and dom manipulations fully within the client browser , this approach is more user friendly as it's lower in latency than the first approach because there is no need for server round trips and full page loads  


In ServiceNow there exists many approaches to building web application for different use cases 


`service portal` or `next experience ui` using ui builder,  or if you have peeking eyes, there also exists a not so common way to build pages using what they call `ui pages`, and we will briefly talk about each of these right now

---

* Service Portal


it's a development platform inside ServiceNow which features angularJS as the backing front-end technology and provides other wrappers to build different backend logic in different dedicated scripts , but given the fact that angularJS is an old technology and is even deprecated by google , servicenow chose to continue supporting it  for backward compatibility and as it's one arm of a whole suite of toolings in service portal to build an application from end to end 

* Next Experience UI 

well stay tuned for a whole LOT of blogs specially dedicated to the idiosyncrasies of this platform and the ui builder tool which is built on top of the web component standard but still lacks a lot of potential to suffice as a mature platform for building mature and sophisticated web applications, can't wait for my awesome friends to amazingly shock you soon enough


* UI Pages

> This is the main focus of this first blog in the series, so let's dive into it 

ui pages is a way to build well - ui pages - in ServiceNow provided their html css, and js content which you can insert inline or import from somewhere else

 not only that , ui page content runs over a template engine called [Apache Jelly ](https://commons.apache.org/proper/commons-jelly/)

 This template engine has certain rules that you must follow for the code to compile as you intend, 
 You will find at the reference section of this blog a youtube video showing a hands-on example approach of how to use jelly directives and syntax


as it's a server side template engine, you typically get access to server side logic and functionality out of the box in their relevant context which allow for seamless integration between the ui content and the empowering server data



now, someone might say, this is 1 ui page , but our goal is to build a full application of so many pages/routes and ui interactions with reactivity and complex data flow

If we refer back to the start of our discussion of types of web apps, specifically the 2nd type , SPA or single page applications, you can read more about them in [here](https://developer.mozilla.org/en-US/docs/Glossary/SPA) , but the interesting concept we pick from SPAs is that they go through build steps , after which only the html css, and js output artifacts that we need are emitted, at which point , we upload these artifacts as that 1 ui page ,and voila,  you have the full blown SPA as you originally developed it, but minified, optionally obfuscated ,and also optimized. Sounds slick and indeed it is! 


So, that's our plan and that's one way of building modern web applications via a SPA frontend technology and for fetching content from the backend we use different APIs protocols and schemes


There are a bunch of SPA frontends in the market developed by big tech companies and used by many others, in this blog we chose to pick react, a fairly common and easy to use library with the its flexibility of Configuration over convention (CoC) design principle to allow developers to choose whatever they want to implement their full product

So, are you excited to see how this could be achieved? so let's begin...

---

In this blog we will follow this medium guide [React in ServiceNow applications (advanced)](https://pishchulin.medium.com/react-in-servicenow-applications-advanced-3e1966fbb817) and break it down step by step as we go through it and explain how things are the way they are



To fully understand the inner workings of this setup , we will need to know some information about  `web bundlers` 


![]({{ site.baseurl }}/images/modern-web-dev/react/wb.png)



web bundlers are a piece of software designed to take as input all assets used in web development like js files, images , css files, fonts, and others and pack them into a unified output artifacts which references all these different types of assets and is production ready to be deployed on a CDN or [Content Delivery Network](https://en.wikipedia.org/wiki/Content_delivery_network) for customers to use 

web bundlers offer a bunch of other features like lazy loading , dead code elimination , shimming, css splitting and others, and they have their own science to tackle and be more experienced about , but now we proceed

As with many things in software engineering, there are many types of bundlers in the wild , each with its pros and cons, examples are `vite`, `Parcel` , and `Webpack`

> Our setup uses webpack so will focus on it


webpack reads a configuration file the developer writes which defines how the web application will be built, in the following image we find the folder storing these configuration in this setup 


![]({{ site.baseurl }}/images/modern-web-dev/react/b1.png)




the config file follows an expected data structure with configuration value to populate and for our case we pinpoint these 2 configuration which are the entry point to our application in src/index.js from which webpack will scrape dependencies and resolve their sources in the dependency graph

and the devServer configuration which comes built in in webpack to allow for smoother development and faster reload, and the rest of the file contains loaders & plugins to merge the assets to bundles as we briefly mentioned


![]({{ site.baseurl }}/images/modern-web-dev/react/b2.png)





Now we follow that entry js script, which renders the application root component defined in App.js , and we could define any react component of our pleasing, but before we do , we will pause for a bit and recall the concept we discussed about backend communication through APIs

We will be using servicenow as our CDN and backend server at the same time for its api offerings 

,and we will use once again, `scripted rest APIs`. We used this feature quite a bit so far for different purposes in our wiki, so it might be a good idea to read our other blogs and enjoy having the sky as your limit 

In fact, we won't write a new scripted rest api from scratch ,and we will reuse one we created and used in our previous blog [A Guide To Achieve Centralized & Automated UpdateSets Workflows](http://snow-wiki.com/blog/toolings/updateset/)

we grab the url of that api endpoint and we expose it for our demo purpose, and as the following screenshot , the api is a post api which sends us some servicenow server side instance information using the glide system api



![]({{ site.baseurl }}/images/modern-web-dev/react/rpc.png)




back to our component , we install an http client in react called `axios` via this command 
`npm i axios`

and insert the following code in `App.js` and the styles in `App.css`.

```jsx

import React , {useState} from "react";

import { hot } from "react-hot-loader/root";

import './App.css'

import axios from 'axios';

  
  
  

function App() {

  const [token, setToken] = useState('');

  const [message, setMessage] = useState('');

  const [currentApp, setCurrentApp] = useState('');

  const [isInteractive, setIsInteractive] = useState(false);

  const [isLoggedIn, setIsLoggedIn] = useState(false);

  

  const fetchData = async () => {

    try {

      const response = await axios.post('https://dev146231.service-now.com/api/x_982379_test_app/rpc', {

        // Add any request payload if needed

      }  , {

  

        headers: {

          'Access-Control-Allow-Origin': '*',

          // Add any other headers as needed

        }

  
  

      });

      const { message : message_content , data } = response.data.result;

  
  
  

      setToken(data.token);

      setMessage(message_content);

      setCurrentApp(data.currentApp);

      setIsInteractive(data.isInteractive);

      setIsLoggedIn(data.isLoggedIn);

  
  

    } catch (error) {

      console.error('Error fetching data:', error);

    }

  };

  

  return (

    <div className="container">

            <h1 className="header">ServiceNow Team is Awesome</h1>

  

      <div className="card">

        <button onClick={fetchData}>Fetch Data</button>

        <div>

          <h2>Token: {token}</h2>

          <p>Message: {message}</p>

          <p>Current App: {currentApp}</p>

          <p>Is Interactive: {isInteractive.toString()}</p>

          <p>Is Logged In: {isLoggedIn.toString()}</p>

        </div>

      </div>

    </div>

  );

}

  
  

export default hot(App);
```



And here is what the ui will look like




![]({{ site.baseurl }}/images/modern-web-dev/react/ui.png)



---


From here, we can use our react skills to build full blown react apps and integrate as we please with servicenow backend or any backend for that matter 




Now, it's time to deploy the react SPA to our ServiceNow instance as a ui page , so we will follow these steps 


1. Build & bundle the react app using webpack 

2. Upload the files into ServiceNow

3. Create a ui page and reference the uploaded assets 

so let's discuss each step further...

1. build & bundle the react app using webpack 

the command to run the build is found in package.json file and is run through `npm run build`

as seen in the following screenshot


![]({{ site.baseurl }}/images/modern-web-dev/react/buildScript.png)




The script removes the old build and uses webpack to build a fresh one with production configuration, the differences in this production config is for optimized and code minification and other things as we mentioned before, the output build is stored in the dist folder as we specify in webpack config in the following image



![]({{ site.baseurl }}/images/modern-web-dev/react/b5.png)



and here is an image of the output artifacts 


![]({{ site.baseurl }}/images/modern-web-dev/react/b6.png)



now you might ask, where is this output structure defined, and it's a great question and a very important follow up

the answer lies in the `postbuild.js`  nodejs script which as named , runs after the build command is finished, so let's take a look into it and pinpoint the important sections to us

```js

function injectAuthLogic(inputHTML) {

  const AUTH_LOGIC_CODE = `<!-- handle security token for API requests -->

  <div style="display:none">

    <g:evaluate object="true">

      var session = gs.getSession(); var token = session.getSessionToken(); if

      (token=='' || token==undefined) token = gs.getSessionToken();

    </g:evaluate>

  </div>

  <script>

    window.servicenowUserToken = '$[token]'

  </script>

  <!-- handle security token for API requests -->

  `

  

  const headEndIndex = inputHTML.indexOf('</head')

  

  return (

    inputHTML.substring(0, headEndIndex) +

    AUTH_LOGIC_CODE +

    inputHTML.substring(headEndIndex)

  )

}

  

function injectJellyDoctype(inputHTML) {

  const DOCTYPE_JELLY = `<g:evaluate>

      var docType = '&lt;!DOCTYPE HTML&gt;';

  </g:evaluate>

  <g2:no_escape>

      $[docType]

  </g2:no_escape>

    `

  

  const headIndex = inputHTML.indexOf('<head')

  

  return (

    inputHTML.substring(0, headIndex) +

    DOCTYPE_JELLY +

    inputHTML.substring(headIndex)

  )

}

  

function removeHtmlTags(inputHTML) {

  return inputHTML.replace(/(<html>)|(<html.+>)/, '').replace('</html>', '')

}

  

function removeDocType(inputHTML) {

  return inputHTML.replace('<!DOCTYPE html>', '')

}
```


As you see from an overview of the script, it contains helper functions to manipulate the built output and transform it in a way that fully conforms to the apache jelly template engine rule set for the content to be correctly rendered as a ui page in ServiceNow


2. upload the files into ServiceNow

Now to the most exciting part of this whole blog , to upload the output artifacts into ServiceNow, guess what, we will use once again, scripted rest APIs and this time for a whole new purpose as a  proxy server from the the ServiceNow instance into the ui page , effectively making it a download dropper endpoint for the ui page,  sounds fun right ? but how so ? 


attached to the medium guide is an updateset file which conveniently installs a mini application with 2 modules, the ui page of the react application and the proxy dropper as a scripted rest api


![]({{ site.baseurl }}/images/modern-web-dev/react/b7.png)



we look at the rest api module and examine it in the following image



![]({{ site.baseurl }}/images/modern-web-dev/react/b8.png)




so we have 3 dropper endpoints for jS files, img files and other files, let's examine the JS endpoint 


![]({{ site.baseurl }}/images/modern-web-dev/react/b9.png)






As we can see in the image above that we grab the filename of the output asset to serve and look it up as an uploaded attachment in the current scripted rest api record whose sys_id is stored in currentRecord variable,  and then we retrieve that attachment and serve as based on its filetype which contains the type extension , then serve that response type as it has to be , hmmm interesting indeed 

but how to attach/upload these assets to the scripted rest api? , maybe looking at the table list view might be a good idea, so we navigate to the table `sys_attachment.list` to find no ui actions there to add new attachments as the image below 



![]({{ site.baseurl }}/images/modern-web-dev/react/b10.png)




, but going back to the form view of the scripted rest api we find that sneaky attachment button there , and so we upload all our assets as shown 


![]({{ site.baseurl }}/images/modern-web-dev/react/b11.png)




Now it's time to add the index.html file in the ui page and reference these assets by their name , but worry not, we won't modify anything as we already built the structure for our ui page dynamically by the `postbuild.js` nodejs script back at build time , so we copy paste index.html from the built assets into the ui page html content 


and now , to the moment of truth, we click the Try it button on the ui page to see our react application shining inside servicenow 


#### Final React App Deployed in ServiceNow


![]({{ site.baseurl }}/images/modern-web-dev/react/app.gif)




> Challenge: How about implementing something like this?



![]({{ site.baseurl }}/images/modern-web-dev/react/challenge.gif)



#### Thanks so much for reading this far and see you in another blog post 👋


---



> References


### Servicenow UI Pages Jellyscript


{% include embed/youtube.html id='HpEaO8DdEV0' %}

