---
layout: post
title: "A Guide To Achieve Centralized & Automated UpdateSets Workflows"
permalink: "updateset"
categories: [Platform Engineering, ServiceNow]
tags: ["r&d" , "python" , "js" , "scripted_rest_apis" , "webhooks" , "reverse_engineering", "automation"]    # TAG names should always be lowercase
image: /images/updateSets/updateSets.jpg


---






> In this blog we are going to live an interesting adventure in which we will use various techniques to reach our goal, and we will be describing our thought process along the way


## Intro
 
 As we develop applications in the ServiceNow platform, we make changes all the times, and these changes are saved into what they call update sets, these sets are stored in xml format and are moved between various environments to apply application configuration and logic


 In this blog , our goal is to provide a way to easily export ,and store these updatesets artifacts into a centralized location for our team to reference back & easily download them from within the servicenow platform from a log table with reference urls from our centralized store, so let's begin! 

## Architecture overview


![]({{ site.baseurl }}/images/updateSets/architecture.png)



we start from our local dev machine by running a script to easily download our target updateset , and then push it to our central repo, we will be using `gitlab` as an example here, after that a webhook we configure in our repo will be triggered pushing the updates to the repo to our snow instance for us to view as records in a logging table, we choose the incident table in this example, with data of the pushed updateset and its download url 

now onto the script contents...

First thing to come to mind, is how we could automate or script out update set export? 

after some research we found that a url could be tweaked with some query parameters to get the job done, the script is as follows: 


```python 

import requests

# Replace the placeholders with your ServiceNow instance URL, username, and password

instance_url = ''

username = ''

password = ''


# Replace the placeholder with the sys_id of the updateset you want to export

updateset_sys_id = ""

  
# Construct the URL for the "Export to XML" related link

export_url = f"{instance_url}/sys_remote_update_set.do?XML&sys_id={updateset_sys_id}"


# Send a GET request to the export URL with basic authentication and stream the response

response = requests.get(export_url, auth=(username, password))

  

# Check if the request was successful

if response.status_code == 200:

    # Save the XML content to a file

    with open("updateSet.xml", "wb") as file:

        for chunk in response.iter_content(chunk_size=8192):

            file.write(chunk)

        print("Updateset exported successfully.")

else:

    print("Failed to export updateset.")
```


Sounds simple, efficient and gets the job done ,but sadly it doesn't 🥺

Our great dev team is currently working on a very awesome & sufficiently large project whose updateset file is quite large `currently 31 MB large`, the above method was found not working to download such sample, given us a small false positive updateset file, not reflecting the original whatsoever. In the following screenshots we show the size difference between the original file & the broken one


![]({{ site.baseurl }}/images/updateSets/1.png)


![]({{ site.baseurl }}/images/updateSets/2.png)



so, there has to be another way , right? We continue digging...

we find the js code of the ui action responsible for exporting the updateset `Export to XML`  ,and we looked it up in the `Local Update Sets` module to find the following:

![]({{ site.baseurl }}/images/updateSets/3.png)



hmm, it invokes a script include and then redirects to a url with some dynamic values

After digging into the script include we found how servicenow's workflow for exporting update set is done:

1. first, during working on projects, updates are pushed to a log of updateset changes in `sys_update_set` table
2. then, when we click this ui action, a clone of the update set in `sys_update_set` is pushed to another table , the `sys_remote_update_set` table, and the updateset state is set to `loaded` and ready to be exported

If we lookup our updateset record there we find another similar `Export to XML` ui action, but this time, this one does the real job we want as stated in the hint field in the following image...


![]({{ site.baseurl }}/images/updateSets/4.png)


If we look at the code we find that, again, a request is made to a specific route with some dynamic parameters to inject, and the couple lines at the end are client-side code to dispatch the whole process, so i think we are ready to start writing our real working script, but first, let's talk about the dynamic parameters to inject 

As shown in the code, the parameters are `sysparm_sys_id` and `sysparm_ck` , and the others are static, but don't let servicenow fool you, this is not the whole picture

To figure this out, we fire up [Burp Suite Community](https://portswigger.net/burp) - An excellent client proxy that intercepts browser requests and possibly modifies, smuggles, or filters them 


after configuring burp suite as our proxy, we click the ui action again and investigate the request as sent by servicenow's own infrastructure,  we find the following viable information


![]({{ site.baseurl }}/images/updateSets/5.png)


as seen in the above image, to establish a successful request , there are other factors to the game , typically request cookies and headers we need to establish before dispatching our request , luckily for us, we can easily do that using burp by copying the request as a curl command and modify the sent headers in a dictionary format as we chose our scripting language for this example to be `python` , with its great `requests` module to automate sending requests and getting responses. 



But about the cookies, how we get them? 

The answer to this question gets us back to a previous blog we wrote in our site here about using [Servicenow REST-API based Automation](https://prod-snow-wiki-9f8c680c23f2905ee2f89363bfbd4da83e3752f335141a01.gitlab.io/blog/hacks/hack1/) module to automate servicenow workflows, so you might want to review that blog if you didn't already before you proceed. 

So as we said, we can construct a servicenow client object into which we store session and cookies information that we can easily retrieve for other use cases, and also we will use a utility from our previous blog titled [Utilities #1](https://prod-snow-wiki-9f8c680c23f2905ee2f89363bfbd4da83e3752f335141a01.gitlab.io/blog/utilities/1/) to get the sys_id of our target updateset which is named `Training and Certification Management` , _I bet you know a little now about our awesome secret project ,but i bet you could keep up with us 😁_ 


The following code does the mentioned job


```python 
import pysnow

c = pysnow.Client(instance="", user="admin", password="")


def get_sys_id(
    table="sys_remote_update_set",
    field_name="name",
    field_value="Training and Certification Management",
):

    qb = (
        pysnow.QueryBuilder()
        .field(field_name)
        .equals(field_value)
        .AND()
        .field("state")
        .equals("loaded")
    )

    callers = c.resource(api_path=f"/table/{table}")

    response = callers.get(query=qb)

    return response.all()[0]["sys_id"]
```


> Note that we used the `sys_remote_update_set` table as we mentioned above and not `sys_update_set`
{: .prompt-tip }



so we have our target updateset sys_id
and we get the cookies easily as follows: 
`cookie_dict = c.session.cookies.get_dict()`


>  Spoiler Alert: we won't need this `cookie_dict` in the end
{: .prompt-tip }


and we got the headers from burp as follows: 

```python 
headers = {
    "Host": {protocolless_instance_url},
    "Sec-Ch-Ua": '"Chromium";v="109", "Not_A Brand";v="99"',
    "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9",
    "Sec-Ch-Ua-Mobile": "?0",
    "Upgrade-Insecure-Requests": "1",
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.5414.120 Safari/537.36",
    "Sec-Ch-Ua-Platform": '"Windows"',
    "Sec-Fetch-Site": "same-origin",
    "Sec-Fetch-Mode": "navigate",
    "Sec-Fetch-Dest": "empty",
    "Referer": "https://dev146231.service-now.com/sys_remote_update_set.do?sys_id={target_updateSet}",
    "Accept-Encoding": "gzip, deflate",
    "Accept-Language": "en-US,en;q=0.9",
    "Connection": "close",
}

```


and we dispatch the request as follows: 

```python 
base_url = f'https://{instance_name}.service-now.com'
url = f'{base_url}/export_update_set.do'
requests.get(url, headers=headers )
```


But what about the request query parameters? we said we have 2 of dynamic values 
`sysparm_sys_id` and `sysparm_ck` 


![]({{ site.baseurl }}/images/updateSets/6.png)


for `sysparm_sys_id` , this is the updateSet sys_id that we got -easy job- but what about the 2nd one `sysparm_ck`!? 

Well, this is when it gets a bit trickier than you might think, after doing some research, this parameter value is identified to be the server session token that servicenow uses for server side operations, now, how to get this value then? following the servicenow developer documentation of the `GlideSession` object at this [Docs](https://developer.servicenow.com/dev.do#!/reference/api/utah/server/c_GlideSessionScopedAPI)

we come about this curious method `getSessionToken()` stating that it `Returns the session token.` ,matching exactly what we want! but how to invoke this server side method from our python code?! this is a backend session data which is not directly accessible with utilities like `pysnow` or `requests` modules as it gets created ,stored ,and maintained by the servicenow instance and is not publicly accessible.


So, after digging deeper we come across a great feature in servicenow called `Scirpted REST APIs` which allows developers to write their own custom rest apis in the platform to perform various functions, we actually use this feature to perform 2 functions in this example, the first of which is right now as we chose to script out a scripted rest api as an [RPC](https://en.wikipedia.org/wiki/Remote_procedure_call) or [Remote procedure call](https://en.wikipedia.org/wiki/Remote_procedure_call) by which we can exfiltrate that sneaky session token that we need to fulfill our request, so we went ahead and created our first scripted rest api named `rpc` and we created a `POST` resource in it , namely we exfiltrate the session token and also possibly other information in the post request body from servicenow that we can remotely invoke via our local python script, sounds great right? and indeed it is :)  , so let's show some code that does exactly that 


![]({{ site.baseurl }}/images/updateSets/7.png)


```js
(function process(/*RESTAPIRequest*/ request, /*RESTAPIResponse*/ response) {
  // implement resource here

  var result = {
    message: "Here is your token!",

    data: {
      token: gs.getSession().getSessionToken(),

      currentApp: gs.getSession().getCurrentApplicationId(),

      isInteractive: gs.getSession().isInteractive(),

      isLoggedIn: gs.getSession().isLoggedIn(),
    },
  };

  response.setContentType("application/json");

  response.setBody(result);
})(request, response);


```

As you saw, we can extract some other useful data outside the platform, so let's proceed...

now we call the RPC from our python script as follows: 

```python 
# Make a POST request to the Scripted REST API

response = requests.post(f"{base_url}{api_path}", auth=auth)


token = None

# Check if the request was successful

if response.status_code == 200:

    response_data = response.json()

    print("rpc output", response_data)

    token = response_data["result"]["data"]["token"]


else:

    print(f"Request failed. Status code: {response.status_code}")

    print(response.text)


sysid = get_sys_id()

# Construct and make the POST request

url = f"{base_url}/export_update_set.do"

parameters = {
    "sysparm_sys_id": str(sysid),
    "sysparm_is_remote": "false",
    "sysparm_ck": str(token),
}

```


now we also have the request parameters and we are ready to go ✌️ 

```python 
response = requests.get(url, params=parameters, headers=headers)

# Check if the request was successful

if response.status_code == 200:

    # Save the XML content to a local file

    with open("updateSet.xml", "wb") as file:

        file.write(response.content)

    print("Export successful! Response XML saved to response.xml.")

else:

    print(f"Request failed. Status code: {response.status_code}")

    print(response.text)
```


If you have been following along thus far, and you ran the script, it will actually fail with a `401 (Unauthorized)` status code indicating that **the request has not been applied because it lacks valid authentication credentials for the target resource** 🥺

The `fix` for this issue is simple but can be easily overlooked, as you recall that http requests are stateless, namely each request has zero knowledge of the previous request , and that's exactly what we have been doing this far in our script by continuously calling  `requests.get()` , `requests.post()` which each initiates a request-response flow from scratch which messes up the session and cookies data we need to have retained across our entire request-response flow, so we fix this by making use of the already established session that got created by the `pysnow` module and following up we dispatch requests on that session object exactly like we did with `requests` as follows: 

```python 
c = pysnow.Client(instance="", user="admin", password="")

session = c.session


# Make a POST request to the Scripted REST API
response = session.post(f"{base_url}{api_path}", auth=auth)

```

And now, we finally can run our script and get a successful exported update_set saved to our current directory, congratulations! 👏

```python 
response = session.get(url, params=parameters, headers=headers)


# Check if the request was successful
if response.status_code == 200:

    # Save the XML content to a local file
    with open("updateSet.xml", "wb") as file:

        file.write(response.content)

    print("Export successful! Response XML saved to response.xml.")

else:

    print(f"Request failed. Status code: {response.status_code}")

    print(response.text)


```


The complete script will be found in this [Repo](https://gitlab.com/ahmed.ibrahim.1sep1997/webhook) for your reference 

--- 

Now, we proceed with our architecture, to the next component , which is setting a gitlab webhook url to point to our snow instance

![]({{ site.baseurl }}/images/updateSets/architecture.png)


On how to setup webhooks on gitlab's repos , refer to this guide at [Gitlab Webhooks](https://docs.gitlab.com/ee/user/project/integrations/webhooks.html)

this web hook is a `push webhook` which we chose to trigger on each & every repo push events to our main branch of our newly exported updateSet.xml file 

to trigger this push, we authored the following python script to automate the pushing process neatly and efficiently, you only need to provide a `.env` file with the following configuration 

```python
GITLAB_REPO_URL=""
GITLAB_ACCESS_TOKEN=""
```

and then simply run the following script which automatically detects new changes to the repo and pushes them to the configured repo 

```python 
import os
import subprocess

from dotenv import load_dotenv

import subprocess

# Load environment variables from .env file
load_dotenv()

# Set the GitLab repository URL
repo_url = os.getenv("GITLAB_REPO_URL")


# Set the path to the current folder
folder_path = os.getcwd()

# Retrieve GitLab access token from environment variables
access_token = os.getenv("GITLAB_ACCESS_TOKEN")


# Change directory to the current folder
os.chdir(folder_path)


# Check if the repository is already initialized
if subprocess.run(
    ["git", "rev-parse", "--is-inside-work-tree"], capture_output=True
).stdout.strip():

    # Repository already initialized, pull changes
    subprocess.run(["git", "pull"])

else:
    # Initialize a new Git repository
    subprocess.run(["git", "init"])

    # Set the GitLab repository URL as the remote origin
    subprocess.run(["git", "remote", "add", "origin", repo_url])
    # Add all files to the repository
    subprocess.run(["git", "add", "."])
    # Commit the changes
    subprocess.run(["git", "commit", "-m", "Automated: New updateSet.xml push commit"])


# Push the changes to the GitLab repository
subprocess.run(["git", "push", "-u", "origin", "main"])

```

after this script is run, and the updateset is pushed to the repo, the webhook is dispatched with a POST request to the configured servicenow instance url , giving in its body information about the repo commits including what files are added or modified and their download url, we use this information later on the servicenow part of the webhook to create incidents inside the platform via a scripted rest api as the second time of using, refer to this [Docs](https://www.servicenow.com/community/in-other-news/how-to-integrate-webhooks-into-servicenow/ba-p/2271745) for more details

We create the 2nd scripted rest api with a post resource to bind to the configured gitlab webhook and named it `webhook` , and following up is the code to create the incidents with their dynamically constructed data combined with a utility we defined in a previous blog in our site `insertRecordsWithReferences(tableName, numRecords, fieldsAndValues) `: 


![]({{ site.baseurl }}/images/updateSets/8.png)



```js

(function process(/*RESTAPIRequest*/ request, /*RESTAPIResponse*/ response) {
  // implement resource here

  var requestBody = request.body.data;

  var added = requestBody.commits.map(function (commit) {
    if (commit.added.length == 0) return;

    var download_url =
      commit.url.split("-")[0] +
      "-/raw/main/" +
      commit.added +
      "?ref_type=heads&inline=false";

    return "Added: " + commit.added + " .To download: " + download_url;
  });

  var modified = requestBody.commits.map(function (commit) {
    if (commit.modified.length == 0) return;

    var download_url =
      commit.url.split("-")[0] +
      "-/raw/main/" +
      commit.modified +
      "?ref_type=heads&inline=false";

    return "Modified: " + commit.modified + " .To download: " + download_url;
  });

  var commitActions = Array.prototype.concat.apply([], [added, modified]);

  function insertRecordsWithReferences(tableName, numRecords, fieldsAndValues) {
    var gr = new GlideRecord(tableName);

    for (var i = 0; i < numRecords; i++) {
      gr.initialize();

      // Set field values based on the input object

      for (var field in fieldsAndValues) {
        if (fieldsAndValues.hasOwnProperty(field)) {
          var value = fieldsAndValues[field];

          // Check if the field is a reference type

          if (
            JSON.stringify(gr[field].getED().getInternalType()) ===
            '"reference"'
          ) {
            var referenceGR = new GlideRecord(value.table);

            if (referenceGR.get(value.sysId)) {
              gr[field] = referenceGR.sys_id;
            }
          } else {
            gr.setValue(field, value);
          }
        }
      }

      gr.insert();
    }
  }

  var tableName = "incident";

  var numRecords = 1;

  var fieldsAndValues = {
    short_description: "New updateSet event!",

    description: commitActions.join("\n\n"),

    priority: 1,
  };

  insertRecordsWithReferences(tableName, numRecords, fieldsAndValues);
})(request, response);

```


And finally here is a screenshot of what the incidents created look like: 


![]({{ site.baseurl }}/images/updateSets/9.png)


> possible next steps
* Implement an Automated updateSet Import & Apply Flow on servicnow instance using the Reverse Engineering Method explained in this blog
* Implement a Desktop GUI to further facilitate the workflow kickoff process



## Outro
And that's all folks, thanks so much if you read up till this point, and see you soon in another blog 👋



