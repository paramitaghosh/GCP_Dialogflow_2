# GCP_Dialogflow_2
deploy a python webhook on AppEngine to handle lookup requests from your HR chatbot

Task 1: Enable APIs
For security purposes, many Google Cloud Services are disabled by default (this is referred to as the principle of least privilege). Ensure the specific APIs needed by this lab are enabled (you can go to APIs & Services > Dashboard and scroll through the list).

Cloud Storage
Cloud Source Repositories API
Cloud Natural Language API
Dialogflow API

Task 2: Initialize App Engine and Cloud Datastore
You'll use a NoSQL database called Cloud Datastore to store content extracted from the document. To enable Cloud Datastore, enable the associated App Engine service.

On the Navigation menu (gcp-menu.png), click App Engine > Dashboard.

Under the Welcome dialog box, click Create Application.

Accept the default region, and then click Create app.

For Language, select Python from the dropdown. You can leave the default, Standard, for Environment.

Click Next to initialize App Engine. It may take a few minutes to initialize. Look for the message box, Your App Engine app has successfully been created, which may be near the bottom of your browser window.

Task 3: Import Entities
Next, import into Datastore the same entities that you created in an earlier lab. For this, we have the files staged in a publically accessible bucket.

From the GCP console, select your project, and then click Activate Cloud Shell( 9db7bdee3b6c113d.png). Click Continue.

In your Cloud Shell, run the following command to import the Topic and Synonym entities into your Datastore.

gcloud datastore import gs://cloud-training/T-GCPDF-I/datastore_HR_entities/2018-11-14T06:19:16_10212.overall_export_metadata
If prompted to Authorize Cloud Shell, click Authorize. After clicking Authorize, you'll need to repeat the previous step to run the command. This will take a minute to complete.

You can verify by going to Navigation menu( 8c24be286e75dbe7.png), click Datastore and select Entities.
Task 4: Getting started with Dialogflow
Dialogflow requires access to a Google account to sign in. You will use the Qwiklabs account that was generated for this lab to sign in and complete the lab. See a list of the permissions and what they're used for.

In a new browser tab, go to https://console.dialogflow.com.

Click on Sign in with Google if prompted.

Task 5. Create a Dialogflow chatbot (Agent)
In this section, you'll create a new virtual agent called hr-bot.

On the Dialogflow welcome page, Click Create Agent on the left.

NOTE: An alternate method is to click the CREATE AGENT button at the lower right.

DF-main-menu.png

In the form, provide the values for the new agent.

Property	Values
Agent Name	Enter a name for your virtual agent. Example: Cloudio
Default Time Zone	Select your timezone. Example: (GMT-8:00) America/Los Angeles
Google Project	Select your GCP Project. Example: qwiklabs-gcp-<random characters>
Click Create.

You'll see the box at the upper right turn green and indicate WORKING until Dialogflow is finished creating your virtual agent.

Task 6: Import your Dialogflow agent
In an earlier lab, you exported the Dialogflow agent you built. You will now import it back in and continue building it.

On the Navigation menu (Navigation menu), click APIs & services.

Scroll down and confirm that your APIs are enabled.

If an API is missing, click ENABLE APIS AND SERVICES at the top, search for the API by name, and enable it for your project.

This will create a new virtual agent project. Now you'll want to import the work you've already done.

Click on the settings gear icon next to your agent name.
DF-agent-settings.png

Select the Export and Import tab.
DF-agent-settings-page.png

Click IMPORT FROM ZIP.

Click SELECT FILE and navigate to the zip file which contains the configuration of your virtual agent. You can alternatively drag and drop the file if you prefer.

DF-upload-agent.png

Type in the word IMPORT in all caps to enable the import button and click IMPORT.
DF-upload-agent-import.png

Click DONE to close out the upload window once the import is complete.
Your existing configuration has been imported into your new agent project.

Task 7: Download the webhook code
In the Google Cloud Console, navigate to the Cloud Shell terminal window.

Run the following commands to create a working directory for your App Engine code.

mkdir hr-chatbot
cd hr-chatbot
Download the chatbot code into your working directory.

gsutil -m cp -r gs://cloud-training/bootcamps/chatbot/code/* .
Task 8: Deploying a Webhook on App Engine
For production you will want your webhook to always be on and ideally able to scale up if your chatbot is popular. You also want your webhook to be available via HTTPS (to protect your data and credentials in transit). In this section, you deploy the Webhook Python code into App Engine, which provides a scalable and secure "always-on" presence for your webhook.

Run the following commands to change into the webhook subdirectory in the code you cloned earlier:

cd webhook
Locate the main.py python file that has been provided and review its content by following the next few steps.
NOTE: You will just be referencing the python code, no need to copy / paste anything. Use nano, vi like editor to review the file.
Note: You are now using the Google Datastore NDB Client Library to allow your App Engine Python app to connect to Cloud Datastore. NDB is the preferred means of interacting with Datastore in App Engine apps, and brings additional capabilities such as Object to Relational Mapping and automatic caching: from google.appengine.ext import ndb
Review the getSynonym() and getActionText() methods, which use NDB to perform Datastore queries.

def getSynonym(query_text):
    synonym_key = ndb.Key('Synonym', query_text)
    synonyms = Synonym.query_synonym(synonym_key).fetch(1)

    synonym_text = ""
    for synonym in synonyms:
        synonym_text = synonym.synonym
        break

    return synonym_text

def getActionText(synonym_text):
    synonym_text = synonym_text.strip()
    topic_key = ndb.Key('Topic', synonym_text)
    topics = Topic.query_topic(topic_key).fetch(1)

    action_text = ""
    for topic in topics:
        action_text = topic.action_text

    if action_text == None or action_text == "":
        return ""

    return action_text
Review the server_error() method, which logs any 500-level internal server errors to StackDriver.

@app.errorhandler(500)
def server_error(e):
    # Log the error and stacktrace.
    print e
    return 'An internal error occurred.', 500
Review classes for Topic and Synonym. These classes inherit from the Model class, which represents the entities stored in Cloud Datastore. NDB uses these classes to map data returned from Datastore into Python classes.

class Topic(ndb.Model):
    action_text = ndb.StringProperty()

    @classmethod
    def query_topic(cls, ancestor_key):
        return cls.query(ancestor=ancestor_key)

class Synonym(ndb.Model):
    synonym = ndb.StringProperty()

    @classmethod
    def query_synonym(cls, ancestor_key):
        return cls.query(ancestor=ancestor_key)
Install all required libraries (including Flask) into a "lib" directory to be deployed with your app to App Engine.

Review the contents of requirements.txt and appengine_config.py, which will ensure that this setup set is completed properly.

Execute the following on the cloud shell:

pip install -t lib -r requirements.txt
Note: If you are getting mssql-cli 1.0.0 has requirement click<7.1,>=4.1, but you'll have click 7.1.2 which is incompatible. as an error, please ignore it.
Review the app.yaml file, which gives App Engine instructions on how to deploy your Python app.

runtime: python27
api_version: 1
threadsafe: true

# [START handlers]
handlers:
- url: /static
  static_dir: static
- url: /.*
  script: main.app
# [END handlers]
To deploy your Dialogflow webhook service to App Engine, execute the following command from your current directory (~/webhook).

gcloud app deploy
Answer Y, when prompted, to continue.

When the command completes, in the GCP Console, on the Navigation menu( 8c24be286e75dbe7.png) click App Engine > Services.

Note that your webhook service is listed as default.

Click on the default hyperlink.
A new tab opens with a Not Found error. Thats OK.

Copy the appspot.com URL for your dialogflow service from your browser's address bar into your clipboard. The appspot.com URL is the URL of the default hyperlink you clicked on the previous step."

Return to the Dialogflow console. On the main menu, click Fulfillment and under the Webhook section, slide right on DISABLED to enable the webhook.

Paste the new App Engine URL for your webhook into Dialogflow, ensuring that it ends with /webhook/.

Note: You must have the leading and trailing slash around webhook.

Click Save to save your webhook configuration.

Return to the Dialogflow test console and test out a phrase in Try it now, such as annual salary or hours of work.

If it works, you have successfully ported your webhook to App Engine! Congratulations! If it doesn't work, retrace your steps to determine what went wrong.

Task 9: Export your agent
In this section, you will export your agent as a zip file so that you can import it later when you start the next lab. This way you can reuse the intents and entities you've configured so far.

Click on the settings gear ⚙ icon next to your agent name in the left menu.
DF-agent-settings.png

In the settings page that opens up, go to the Export and Import tab.
DF-agent-settings-page.png

Click on EXPORT AS ZIP. This will download your agent into a local zip file.
