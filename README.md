# Updating the Send Message Functionality

## Learning Goals

- Refactor message sending functionality.

## Introduction

We have an issue with the "send message" functionality now. If you use your
application to send a message, you will see that the list of user messages no
longer updates. That's because we now get the user messages from our new API, so
when a new message gets added locally, it then gets overridden by the content
coming back from the API.

In other words, we need to implement the functionality to implement the API
endpoint for the "send" functionality as well.

Great - this will give us an opportunity to practice another `HTTP` verb:
`POST`.

## Refactor

We use `POST` when we want to write data to our API, but before we can call a
new API endpoint, we have to create it on the Java side - here is the new
version of our `MessagingController`:

1. First, let's update our `SecurityConfiguration` to disable CSRF checks - this
   may not be something you want to do so generally in a real-life application,
   but it works fine for our simple example here

```java
package com.flatiron.spring.FlatironSpring;

import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;

@Configuration
@EnableWebSecurity
public class SecurityConfiguration  extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                .anyRequest().permitAll().and().csrf().disable();
    }

}
```

2. Then we modify our `MessagingController` to add a new endpoint:
   1. Since we are now updating and reading the `userMessages` and
      `senderMessages` lists, we need to make them global so that both the
      `get()` and `add()` methods can access the same values
   2. We create a method to initialize our new global `userMessages` and
      `senderMessages` objects. The `@PostConstruct` annotation tells Spring to
      call the `initMessages` method after the instance of the controller class
      has been instantiated
   3. The `getUserMessages()` and `getSenderMessages()` then become very simple
      methods that just return the current values for their respective lists
   4. We create the actual endpoint and simply add the `Message` object we
      receive to the existing `userMessages` list
3. Here is the updated `MessagingController` class:

```java
package com.flatiron.spring.FlatironSpring;

import org.springframework.web.bind.annotation.*;

import javax.annotation.PostConstruct;
import java.util.ArrayList;
import java.util.List;

@RestController
public class MessagingController {

    List<Message> userMessages = new ArrayList<Message>();
    List<Message> senderMessages = new ArrayList<Message>();

    @PostConstruct
    private void initMessages() {
        userMessages.add(
                new Message(
                        new User("Aurelie"),
                        "Message from Lilly",
                        1, 2
                )
        );

        senderMessages.add(
                new Message(
                        new User("Ludovic", true),
                        "Message from Ludo",
                        1, 0
                )
        );
        senderMessages.add(
                new Message(
                        new User("Jessica", false),
                        "Message from Jess",
                        1, 1
                )
        );

    }

    @CrossOrigin(origins = "http://localhost:4200")
    @GetMapping("/api/get-user-messages")
    public List<Message> getUserMessages() {
        return userMessages;
    }

    @CrossOrigin(origins = "http://localhost:4200")
    @GetMapping("/api/get-sender-messages")
    public List<Message> getSenderMessages() {
        return senderMessages;
    }

    @CrossOrigin(origins = "http://localhost:4200")
    @PostMapping("/api/add-user-message")
    public List<Message> addUserMessage(@RequestBody Message newMessage) {
        userMessages.add(newMessage);
        return userMessages;
    }
}

class Message {
    public User sender;
    public String text;
    public int conversationId;
    public int sequenceNumber;

    public Message(User sender, String text, int conversationId, int sequenceNumber){
        this.sender = sender;
        this.text = text;
        this.conversationId = conversationId;
        this.sequenceNumber = sequenceNumber;
    }

}

class User {
    public String firstName;
    public boolean isOnline = false;

    public User(String firstName) {
        this.firstName = firstName;
    }

    public User(String firstName, boolean isOnline) {
        this.firstName = firstName;
        this.isOnline = isOnline;
    }
}
```

Now that we have the `add-user-message` endpoint implemented, we can call it
from the `addUserMessage()` function in `messaging-data.service.ts`:

```typescript
    addUserMessage(newMessage: Message) {
        this.httpClient.post<Message[]>("http://localhost:8080/api/add-user-message", newMessage).subscribe(
            (messages: Message[]) => {
                console.log(messages);
                this.userMessages = messages;
                this.userMessagesChanged.emit(this.userMessages);
            }
        )
        this.userMessages.push(newMessage);
        this.userMessagesChanged.emit(this.userMessages.slice());
    }
```

Your "send message" functionality should now be back up and running.
