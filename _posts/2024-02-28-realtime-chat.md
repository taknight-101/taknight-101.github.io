---
layout: post
title: "A Complete Guide on How to Build A Realtime Chat Application In ServiceNow"
permalink: "realtime-chat"
categories: [Platform Engineering, ServiceNow]
tags: ["graphql" , "websockets" , "ui_builder" , "js"]    # TAG names should always be lowercase
image: /images/realtime/realtime.jpg


---




## _In this blog, We are going to build a real time chat application from start to finish_


I had some goals in mind that i wanted to tackle regarding why to implement such application, so i will list them first and foremost 


Basically i wanted to:
###### 1. Write a blog about using graphql apis in ServiceNow 
###### 2. Research the possibility of using reactive features inside ServiceNow for possible use cases and scenarios
###### 3. Methodize a development process for me and my awesome team for future project engagements employing systems thinking and agile principles based off my personal view which has been forming since my first interaction with the ServiceNow platform , so this blog - and all other blogs of which i'm the author - are totally subjective and can be further improved and optimized for different scenarios, so without further ado ,let's begin the technical talk 

---

As mentioned , we want to build a chat application, and chat apps have the implicit requirement of being interactive and reactive ,i mean who wants to refresh their active browser window each time they send a message to someone or receive one, not only it's considered poor user experience, but totally inefficient and time wasting. 

and here our logical thinking makes us question, how Realtime web applications are implemented to begin with? ,and the answer to that - saving us some history talk we don't need for now - is behind a web protocol called websockets, websocket as mentioned in [MDN](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API) 

**Defines as** 

`an advanced technology that makes it possible to open a two-way interactive communication session between the user's browser and a server. With this API, you can send messages to a server and receive event-driven responses without having to poll the server for a reply.`

Simply put, it's a way to send and receive messages in the same communication channel/socket so as to not lose an already opened connection and have to restart/refresh your browser to initiate a new connection once again, which typically is what we need 

so, it's a web protocol , much like http/s which defines a scheme or a contract of rules both communicating parties have to follow to achieve this low-level construct of duplex communication

for that, there exists so many utilities, wrappers, libraries whose purpose is to implement the interfacing with this protocol in a variety of programming languages and technology stacks, for instance , .net stack uses [signalr](https://dotnet.microsoft.com/en-us/apps/aspnet/signalr), and nodejs uses [socket.io](https://socket.io/), but these are `pro-code` development environments and are platform agnostic, while in our case here, we are using a [PAAS](https://azure.microsoft.com/en-us/resources/cloud-computing-dictionary/what-is-paas) or `platform as a service` being ServiceNow as a layer we have to abide by its implementations and rules to achieve whatever we want to do, so now the question becomes - how to do websockets and Realtime communication inside ServiceNow? 

I did a research on that topic and tried many ways to achieve that goal, to mention a few, i thought about escaping the ServiceNow next-experience sandbox inside ui builder in client scripts to get access to the DOM in order to get hold of the websocket api which is registered in the document object in js by means of `this` keyword re-binding , but that wasn't possible and is heavily restricted and forbidden as the next experience ui framework in ServiceNow implements the web component standard which is a declarative approach into developing frontends by means of state manipulation and components apis , and that was even the client side of websocket communication, for the server side i came across [this support knowledge article](https://support.servicenow.com/kb?id=kb_article_view&sysparm_article=KB0829978) in ServiceNow dictating that ServiceNow actually support websockets in the backend for Realtime actions! and they even use that in the little green checkboxes you see on fields on a record form when someone else is editing it in the tables form view!, so that's great that there is websocket support in ServiceNow ,and after more research, i found that there is a subsystem or feature you could say in ServiceNow called  [AMB or Asynchronous Message Bus](https://docs.servicenow.com/bundle/washingtondc-platform-administration/page/administer/platform-performance/concept/AMB-metrics.html)


which as they mention in the docs is  `available to view on the ServiceNow Performance homepage to use asynchronous communication to monitor the transaction count and response times, which provide a performance analysis.`

hmm, monitor & performance analysis are typical Realtime processes which require fast feedback loop at each update,  so there must be Realtime implementation secrets in there...

After digging deeper about this AMB i came across this blog titled [How to use AMB Record Watcher in ServiceNow UIB](https://www.servicenow.com/community/next-experience-articles/how-to-use-amb-record-watcher-in-servicenow-uib/ta-p/2352975)


interesting, a record watcher that uses amb to get feedback on records and their states , and at the same time in ui builder! Sounds like the full matter is signed off end to end, and indeed it is 😄 . There is a data resource in ui builder with the name `Record Watcher `
which you configure to listen to updates to any tables you wish and be notified about the event that just happened and even have access to the event payload of what has happened, and that's what we want to achieve real time communication inside servcienow as we please, so now, let's get down to the implementation

---

> As mentioned at the beginning of this blog, one of the goals of writing this blog is to methodize the development process of applications in servicenow , but i think it would be better to strip this part into a future dedicated blog to keep track and emphasis on the subtle difference between process management and its concrete implementation, also to keep this blog focused and not too long to read, so let's proceed 


We start by navigating to `app engine studio` for faster application bootstrapping , and went ahead with choosing the name `chat-app` and default 2 roles for admin and user to have access to the application and its features.

Regarding Data modelling of chat apps => one could come up with various design choices with informed justifications , but in our case, we chose 2 tables only to represent our app , and let's pause for a moment and mention briefly the application features to set the scope early in our discussion 


> * The app features 2 data concepts, chat rooms, and chat messages

> * Admin user only creates chat rooms and manage the assignment of members to different rooms for them to chat in

> * Users only see their assigned rooms and nothing else 

and that's it, simple and enough for our purposes

---


The data layer consists only of 2 tables `chat_room` , and `messages`. Following are their schema shapes: 




![]({{ site.baseurl }}/images/realtime/b1.png)

![]({{ site.baseurl }}/images/realtime/b2.png)

![]({{ site.baseurl }}/images/realtime/b3.png)



---

The portal consists also of 2 pages for the relevant data models of rooms and chat messages, following are their visual hierarchy 



![]({{ site.baseurl }}/images/realtime/b5.png)

![]({{ site.baseurl }}/images/realtime/b6.png)


---

And as said, the application uses 2 default roles to be able to use the application and no automation was needed 



![]({{ site.baseurl }}/images/realtime/b7.png)



and that's it for the overview of the app structure, now, let's dig into the details.. 


We start by the first page , the `rooms` page



![]({{ site.baseurl }}/images/realtime/b8.png)

as seen above, the structure is simple, the rooms are represented as boxes and below each room is a button to go to that room and start chatting, the boxes show the room name and assigned members by the admin of the application which does that via the classic ui in the platform



![]({{ site.baseurl }}/images/realtime/b9.png)

![]({{ site.baseurl }}/images/realtime/b10.png)



now, how are these data fetched from the tables?

The answer to this question follows the 1st goal in this blog which is using graphql APIs to fetch backend data. 

The benefits of using graphql instead of REST APIs is substantial and crucial in some cases, and you can read more about graphql, its features and offerings as an alternative option to REST => The official docs is your place which can be found [here](https://graphql.org/)

In short there are `3` key concepts that are required to implement graphql APIs inside servicenow: 

1. The graphql schema
2. Scipted resolvers 
3. Resolver mappings 



![]({{ site.baseurl }}/images/realtime/b11.png)

![]({{ site.baseurl }}/images/realtime/b12.png)



Now, We briefly discuss them: 

1. `The graphql schema` as in any API, is the contract between the caller and the fulfiller, the fields & their types that the requester wishes the backend to return back, and in graphql, this strongly typed schema definition allows for easier and more efficient API versioning which was a huge drawback in its REST APIs counterpart, just to name a few

schemas in graphql have 2 root types, `query and mutation`, the former for querying data as with R in REST's CRUD and the latter for modifying data as with C,U,D in REST operations  

In our application we defined the schema as follows : 

```graphql

schema {
    query: Query
    mutation: Mutation
}

type Query {
   #implement here...
	getRooms : [Room]
}
type Room { 
name: String @source(value: "name.display_value")
	members :  [User] @source(value: "members.value")
	sys_id : ID  @source(value: "sys_id.value")
}

type User {
  id: String! @source(value: "sys_id.value")
  active: String @source(value: "active.display_value")
  name: String @source(value: "name.value")
  email: String @source(value: "email.value")
  manager: User @source(value : "manager.value")
}


type Mutation {
    #implement here...
	    createMessage(from:ID,to:ID , body : String , room_id:  ID ): Message

	
}

type Message { 

from : ID
to : ID 
body : String
room_id : ID
	
}


```

Some notes about the schema:

1. as seen above, we only choose the fields we want for the types we want and this allows for efficient data fetch that leads to faster network roundtrips


2. the  `@source` directive allows for target field specification as in ` active: String @source(value: "active.display_value")` as 
`active : String` alone would return a stringified object containing both value and display_value key-value pairs, 
also it allows us to use aliases for target fieldNames as with `id: String! @source(value: "sys_id.value")`, as there is no column field name in the User table with the name of `id` so we specify that it binds to the underlying sys_id.value in the user table

3. mutations are like methods or remote procedure calls to post to the backend for execution and just like programming languages methods,are parameterized and invoked 


4. to test out these queries and mutations, we installed this application's updateset from [this repo](https://github.com/ServiceNowNextExperience/graphiql)

which exposes the `graphql explorer` module to access the graphiql tool.

This tool provides an easy to use interface to test graphql operations from within the servicenow as seen below 



![]({{ site.baseurl }}/images/realtime/b14.png)


---

2. `Scipted resolvers `

these are the actual server apis implementation to resolve the query the client requests, we created 3 resolvers in the app as we have 3 types to fulfill their data , we describe one of them now and the rest are similar 

```js
(function process( /*ResolverEnvironment*/ env) {

  

    // implement data resolver here

  

    var sysId = env.getArguments().id || env.getSource();

  
  

    var sysIdList = sysId.split(',');

    var gr = new GlideRecordSecure('sys_user');

    gr.addQuery('sys_id', 'IN', sysIdList); // Use 'IN' operator to match multiple sys_ids

    gr.query();

  
  

    return gr;

  

})(env);

```


This is the script that fetches the data from the servcienow tables, and as seen above this resolver is the `Get User` resolver, which handles the `User type` defined in schema as 

```graphql
type User {
  id: String! @source(value: "sys_id.value")
  active: String @source(value: "active.display_value")
  name: String @source(value: "name.value")
  email: String @source(value: "email.value")
  manager: User @source(value : "manager.value")
}

```

The column names defined from the user table use the `@source` directive to bind to the specific values , we note that the `manager` field is a reference field to the user table in itself , and for that we slickly use the same resolver to resolve that field , also the `env.getSource()` and the lines below

```js
 var sysIdList = sysId.split(',');

    var gr = new GlideRecordSecure('sys_user');

    gr.addQuery('sys_id', 'IN', sysIdList);
```

allows for `1:many` type resolution for fields such as `members` in the `rooms` table.


So overall, the script checks if the client passes any arguments to retrieve a specific user and if not, it fetches relevant reference fields and return the data back to the requester 

----

1. Resolver mappings 

the final step which is binding the resolvers to the schema types as seen below 


![]({{ site.baseurl }}/images/realtime/b15.png)



{{< notice "note" >}}
note that the resolver `Get User` is used many times to resolve fields of the same field type in different places 
{{< /notice >}}


now, how to consume these apis from ui builder, the frontend of our application? 

Via a graphql data resource



![]({{ site.baseurl }}/images/realtime/b16.png)



After testing the query in graphiql interface, we copy it to the Query field in the resource definition as below


![]({{ site.baseurl }}/images/realtime/b17.png)


 and that's all,now we can use ui builder's different techniques to bind data resources to components as we wish

in our example we used a repeater to render the rooms that are accessible to the current logged in user as being part of assigned members, following is the repeater js code 

```js
/**

 * @param {params} params

 * @param {api} params.api

 * @param {TransformApiHelpers} params.helpers

 */

  
  

function evaluateProperty({

    api,

    helpers

}) {

  

    let rooms = api.data.chatsgraphqlapi_1.output.data.x982379ChatAp0.chat.getRooms;

  

    let my_rooms = []

  
  
    if (api.context.session.user.userName === "admin") {

        return rooms;

  

    }

  
  
  

    rooms.forEach(r => {

  

        r.members.forEach(m => {

            if (m.id === api.context.session.user.sys_id) {

                my_rooms.push(r)

            }

  
  

        })

    });

  
  
  
  

    return my_rooms;

}
```


{{< notice "note" >}}
 note that there are many ways to implement authorization in servicenow, ACLs are just one way , the above method of using logged in sys_ids and comparing them to other sys_ids is a another programmatic approach, choose what it best fits the problem at hand
{{< /notice >}}


---

now , back again to our 2nd goal in this blog with the 2nd page in which we use AMB to achieve realtime chatting 

as seen below , the page is again simple and concise 




![]({{ site.baseurl }}/images/realtime/b18.png)



it features the current room name, the user messages are associated with the poster name and the date & Time the message was sent at before the message content

the message content is managed by a 2 way data binding scheme in a client state that updates at each input key stroke and after the send button is clicked the data is sent to the backend via a graphql mutation api defined as discussed previously with the slight difference this time as it's being parameterized ,and the way to parameterize a data resource in ui builder is also simple and straight forward, as it's a meta-data defined object in the resource definition with the parameters information you wish to pass on, following is a screenshot of the createMessage mutation api



![]({{ site.baseurl }}/images/realtime/b19.png)


And the meta-data object defined for the parameters of the api are as follows for reference: 

```json

[
  {
    "name": "body",
    "label": "Message",
    "description": "Sent message",
    "fieldType": "string",
    "mandatory": true
  },
  {
    "name": "from",
    "label": "From",
    "description": "From User",
    "fieldType": "ID",
    "mandatory": false
  },
  {
    "name": "to",
    "label": "To",
    "description": "To User",
    "fieldType": "ID",
    "mandatory": false
  },
  {
    "name": "room_id",
    "label": "room_id",
    "description": "Room",
    "fieldType": "ID",
    "mandatory": true
  }
]
```

The api is consumed after the send button is clicked as follows:



![]({{ site.baseurl }}/images/realtime/b20.png)


now with the realtime data resource , the `record watcher` 

> Attached at the reference section of this blog a detailed description of AMB and record watchers

Suffice to say how ours looks like as the message body field changes after the graphql mutation is invoked 



![]({{ site.baseurl }}/images/realtime/b21.png)


![]({{ site.baseurl }}/images/realtime/b22.png)

---

AMB notifies connected active front-ends of the event that happened to the messages table which triggers the realtime broadcast of updates with the event payloads to all receiver, at which point we hook into that event after it's emitted, parse and pick the fields we want from its payload, save the data structure we choose in a client state parameter and render the messages inside the repeater which binds to this client state parameters and voila , goal achieved :) 

Following is the code invoked after the amb event is emitted 

```js
/**

 * @param {params} params

 * @param {api} params.api

 * @param {any} params.event

 * @param {any} params.imports

 * @param {ApiHelpers} params.helpers

 */

function handler({

    api,

    event,

    helpers,

    imports

}) {


  

    let record = event.payload.data.record;

    let body = record.body.value

    let created_by = record.sys_created_by.value

    let at_time = record.sys_updated_on.value

    let room_id = record.room_id.value

  

    let is_current_room = room_id === api.context.props.sysId

  

    if (!is_current_room) {

        return

    }

  

    var date = new Date(at_time);

  

    // Define formatting options

    var options = {

        year: 'numeric',

        month: 'long',

        day: '2-digit',

        hour: '2-digit',

        minute: '2-digit',

        second: '2-digit',

        hour12: true // Set to true to display in AM/PM format

    };

  

    // Format the date

    var formattedDate = date.toLocaleString('en-US', options);

  
  

    let newMessages = [...JSON.parse(api.state.messages || '[]'),  {

        'message': body,

        'from': created_by,

        'time': formattedDate,

        'room_id': api.context.props.sysId

    }]

  
  

    api.setState('messages', JSON.stringify(newMessages));

  
  
  

}

```


{{< notice "note" >}}
  note that we implemented authorization here programmatically again to make sure we don't broadcast messages to rooms they don't belong to, these lines in the below code do the trick 
{{< /notice >}}



```js

   let is_current_room = room_id === api.context.props.sysId

  

    if (!is_current_room) {

        return

    }


```

and this client script accounts for the initial load of the chat page which retrieves all previous messages in the room from participating parties, as being invoked after a data fetch from an api but rather a REST api this time :) 

No hard feelings for you REST APIs authors out there :) Software Engineering is the art of problem solving at the end of the day. 



```js

/**

 * @param {params} params

 * @param {api} params.api

 * @param {any} params.event

 * @param {any} params.imports

 * @param {ApiHelpers} params.helpers

 */

function handler({

    api,

    event,

    helpers,

    imports

}) {

  

    let newMessages = [...JSON.parse(api.state.messages || '[]')]

  
  

    let body = api.data.look_up_records_1.results.forEach(record => {



        if(api.context.props.sysId === record.room_id.value){

                api.setState('current_room' ,record.room_id._reference.name.value )

        }

  
  

        let body = record.body.value

        let created_by = record.sys_created_by.value

        let at_time = record.sys_updated_on.value

  

        newMessages.push({

            'message': body,

            'from': created_by,

            'time': formatDate(at_time)

  

        })

  

    })

  

    function formatDate(time) {

        var date = new Date(time);

  

        // Define formatting options

        var options = {

            year: 'numeric',

            month: 'long',

            day: '2-digit',

            hour: '2-digit',

            minute: '2-digit',

            second: '2-digit',

            hour12: true // Set to true to display in AM/PM format

        };

  

        // Format the date

        return date.toLocaleString('en-US', options);

  

    }

  

    api.setState('messages', JSON.stringify(newMessages));

  
  
  

}

```

---

### 2 Short Demos Of The Final App


![]({{ site.baseurl }}/images/realtime/demo.gif)

---

![]({{ site.baseurl }}/images/realtime/demo2.gif)


---

##### And that's all. Thank you all for reaching this far, and hope you enjoyed this blog as well as i hope to see you soon in another blog 

> References

### 1.graphql

[Setting up and Testing your first GraphQL API Tutorial (Part 1 of 3)](https://www.servicenow.com/community/now-platform-articles/setting-up-and-testing-your-first-graphql-api-tutorial-part-1-of/ta-p/2307775)


[Setting up and Testing your first GraphQL API Tutorial (Part 2 of 3)](https://www.servicenow.com/community/developer-articles/querying-data-with-your-graphql-api-tutorial-part-2-of-3/ta-p/2296493)

[Setting up and Testing your first GraphQL API Tutorial (Part 3 of 3)](https://www.servicenow.com/community/now-platform-articles/using-data-from-your-graphql-api-in-ui-builder-part-3-of-3/ta-p/2315430)


[GraphQL Overview - Learn Integrations on the Now Platform](https://www.youtube.com/watch?v=sP_CelE8GGI&list=PL3rNcyAiDYK0at2ypM6uhp_Mg6-gZlIdP&index=45)


### 2. AMB 


[Docs](https://docs.servicenow.com/bundle/washingtondc-platform-administration/page/administer/platform-performance/concept/AMB-metrics.html)


[How to use AMB Record Watcher in ServiceNow UIB](https://www.servicenow.com/community/next-experience-articles/how-to-use-amb-record-watcher-in-servicenow-uib/ta-p/2352975)

