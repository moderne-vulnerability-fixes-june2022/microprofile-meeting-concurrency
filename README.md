# Part 3: MicroProfile Meeting Application - Using Java EE concurrency

## Overview
In this lab we will look at doing some asynchronous tasks in the application to perform background maintanance.

Until now, when a meeting is started, a link is stored to the e-meeting but never removed when the meeting ends. That means the next time a meeting gets run it will redirect everyone to the old meeting, which is clearly not right. You want to ensure that after the meeting ends the URL gets deactivated. In this article we will use the concurrency utilities for Java EE to achieve this.

Adapted from the blog post: [Writing a simple MicroProfile application: Using Java EE concurrency](https://developer.ibm.com/wasdev/docs/simple-microprofile-app-3-using-concurrency/)

## Prerequisites
 * Completed [Part 2: MicroProfile Meeting Application - Adding persistance](https://github.com/IBM/microprofile-meeting-persistance)
 * [Eclipse Java EE IDE for Web Developers](http://www.eclipse.org/downloads/)
 * IBM Websphere Application Liberty Developer Tools (WDT)
   1. Start Eclipse
   2. Launch the Eclipse Marketplace: **Help** -> **Eclipse Marketplace**
   3. Search for **IBM Websphere Application Liberty Developer Tools**, and click **Install** with the defaults configuration selected
 * [Git](https://git-scm.com/downloads)
 
## Steps
### Step 1. Check out the source code

#### From the command line
Run the following commands:
  
```
$    git clone https://github.com/IBM/microprofile-meeting-concurrency.git
```

#### In Eclipse, import the project as an existing project.
1. In Eclipse, switch to the Git perspective.
2. Click **Clone a Git repository** from the Git Repositories view.
3. Enter URI `https://github.com/IBM/microprofile-meeting-concurrency.git`
4. Click **Next**, then click **Next** again accepting the defaults.
5. From the **Initial branch** drop-down list, click **master**.
6. Select **Import all existing Eclipse projects after clone finishes**, then click **Finish**.
7. Switch to the Java EE perspective.
8. The meetings project is automatically created in the Project Explorer view.

### Step 2. Installing MongoDB
If you completed the previous labs and installed MongoDB, make sure MongoDB is running. If you are starting fresh, make sure you install MongoDB. Depending on what platform you are on the installation instructions may be different. For this exercise you should get the community version of MongoDB from the [mongoDB download-center](https://www.mongodb.com/download-center#community).

1. Once installed you can run the MongoDB database daemon using:
```bash
mongod -dbpath <path to database>
```

The database needs to be running for the application to work. If it is not running there will be a lot of noise in the server logs.

### Step 3. Updating the application to compile against the Concurrency API for Java EE
To start writing code the Maven `pom.xml` needs to be updated to indicate the dependency on the Concurrency API for Java EE:

1. Open the `pom.xml` in Eclipse.
2. In the editor, select the **Dependencies** tab.
3. On the **Dependencies** tab there are two sections, one for Dependencies and the other for Dependency Management. Just to the right of the Dependencies box there is an **Add** button. Click the **Add** button.
4. Enter a **groupdId** of `javax.enterprise.concurrent`.
5. Enter a **artifactId** of `javax.enterprise.concurrent-api`.
6. Enter a **version** of `1.0`.
7. From the `scope` drop-down list, select **provided**. This will allow the application to compile but will prevent the Maven WAR packager putting the API in the WAR file. Later, the build will be configured to make it available to the server.
8. Click **OK**.
9. Save the `pom.xml`.

### Step 4. Updating the meeting manager to end the meeting
1. Open the `MeetingManager` class from: **meetings > Java Resources > src/main/java > net.wasdev.samples.microProfile.meetings > MeetingManager.java**.

2. Just below the class definition, just after the DB definition, get a `ManagedScheduledExecutorService` injected:
```java
@Resource
private ManagedScheduledExecutorService executor;
```

3. This introduces a new class, the `ManagedScheduledExecutorService`, which is in the `javax.enterprise.concurrent` package:
```java
import javax.enterprise.concurrent.ManagedScheduledExecutorService;
```

4. Find the `startMeeting` method. This is where we will put the logic for ending the meeting. Place all the code at the very end of the method. The event will be scheduled to run, and a response isn’t required because we just need to ensure it happens later on. Multiple steps will take place before all compile errors will be gone. The code will call the `schedule` method of the injected `ManagedScheduledExecutorService`, this requires a Runnable, a duration, and a time unit for the duration.

* First, let’s get the meeting duration from the database. It’ll be a `Long` in the database but the `get` method just returns `Object`. If we cast it to a `number` and then call `longValue` we get a `Long` and will be resilient if the code later gets changed to add an `Integer` instead:
```java
long duration = ((Number)obj.get("duration")).longValue();
```

* Next, we need to define a time unit. In reality, meeting durations are measured in minutes or hours. The correct code would be:
```java
TimeUnit unit = TimeUnit.MINUTES;
```

but this would not be very useful for demo/sample purposes so instead add:
```java
TimeUnit unit = TimeUnit.SECONDS;
```

* This introduces a new class `TimeUnit` which is in the package `java.util.concurrent`:
```java
import java.util.concurrent.TimeUnit;
```

* In the next step you will add a `run` method in which you will need to find the entry that needs to have the meeting removed. To do this we need access to the `id` variable in the enclosing method. To access the `id` variable, it needs to be marked `final`. So update the first variable declaration in the `startMeeting` method to be:
```java
final String id = meeting.getString("id");
```

* Call the `schedule` method of the injected `ManagedScheduledExecutorService` passing in an anonymous inner class of type `Runnable`:
```java
executor.schedule(new Runnable() {
     
    @Override
    public void run() {
        // code will be added here
    }
}, duration, unit);
```

* As with all things involving the data model, the first thing to do is get the MongoDB collection. This code should go in the `run` method that you added above:
```java
DBCollection coll = getColl();
```

* Next, get the `DBObject` representing the meeting:
```java
DBObject obj = coll.findOne(id);
```

* Resetting the meeting URL is as simple as just removing the field:
```java
obj.removeField("meetingURL");
```

* Then it needs to be saved back to MongoDB:
```java
coll.save(obj);
```

* Everything is done but, since these things are asynchronous for debug purposes, it is a good idea to write something to the log. You could use a proper logging API but, in this case, `System.out` is good enough. At the end of the `run` method, add:
```java
System.out.println(id + " meeting ended");
```

5. Save the file.

### Step 5. Configuring Liberty to run the Concurrency Utilities for Java EE
1. Open the `server.xm`l from **src > main > liberty > config > server.xml**.

2. Find the feature manager element. It should look like this:
```java
<featureManager>
    <feature>mongodb-2.0</feature>
</featureManager>
```

3. Before the closing `</featureManager>` element add a feature element with the feature `concurrent-1.0` as the body.
```java
<feature>concurrent-1.0</feature>
```

4. Save the file.

### Step 6. Ensuring the Concurrency Utilities for Java EE are available
The Concurrency Utilities for Java EE are not part of the Java EE Web profile, which means that if you are using that distribution of Liberty the feature won’t be available. The `pom.xml` generated by the [Liberty app accelerator](https://liberty-app-accelerator.wasdev.developer.ibm.com/start/) depends on the Java EE Web profile distribution, so you need to either install `concurrent-1.0` from the Liberty Repository or pull in the larger Java EE Full platform distribution:

1. Open the `pom.xml`.

2. Switch to the `pom.xml` tab in the editor.

3. Then take one of the following approaches:

* To switch to the Java EE Full platform distribution of Liberty search for `wlp-webProfile7` and change it to `wlp-javaee7`. After this change you should see this:
```xml
<assemblyArtifact>
    <groupId>com.ibm.websphere.appserver.runtime</groupId>
    <artifactId>wlp-javaee7</artifactId>
    <version>17.0.0.1</version>
    <type>zip</type>
</assemblyArtifact>
```

* To download the just `concurrent-1.0` feature, search for `mongodb-2.0` and add another feature element but, this time, with `concurrent-1.0` mentioned. After this change you should see this:
```xml
<features>
    <acceptLicense>true</acceptLicense>
    <feature>mongodb-2.0</feature>
    <feature>concurrent-1.0</feature>
</features>
```

4. Save the file

### Some final thoughts
Those of you with eagle eyes will notice that there is a flaw in this logic: It isn’t very fault-tolerant. If I have a 90-minute meeting and the server crashes or restarts, the meeting will still never end. There are ways (he writes mysteriously) to solve this problem but that is an exercise for another day.

### Running the application
There are two ways to get the application running from within WDT:

 * The first is to use Maven to build and run the project:
 1. Run the Maven `install` goal to build and test the project: Right-click **pom.xml** in the `meetings` project, click **Run As… > Maven Build…**, then in the **Goals** field type `install` and click **Run**. The first time you run this goal, it might take a few minutes to download the Liberty dependencies.
 2. Run a Maven build for the `liberty:start-server goal`: Right-click **pom.xml**, click **Run As… > Maven Build**, then in the **Goals** field, type `liberty:start-server` and click **Run**. This starts the server in the background.
 3. Open the application, which is available at `http://localhost:9080/meetings/`.
 4. To stop the server again, run the `liberty:stop-server` build goal.

 * The second way is to right-click the `meetings` project and select **Run As… > Run on Server** but there are a few things to note if you do this. WDT doesn’t automatically add the MicroProfile features as you would expect so you need to manually add those. Also, any changes to the configuration in `src/main/liberty/config` won’t be picked up unless you add an include.

Find out more about [MicroProfile and WebSphere Liberty](https://developer.ibm.com/wasdev/docs/microprofile/).

## Next Steps
Part 3: [MicroProfile Meeting Application - Using WebSockets and CDI events](https://github.com/IBM/microprofile-meeting-websockets)
