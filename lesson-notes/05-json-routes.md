## AngularSpring

Create a project via Spring Initializr called `AngularSpring` with the following libraries:

* Web
* DevTools
* JPA
* H2

Download [angular.zip](https://github.com/oakes/java-assignments/raw/master/curriculum/assets/angular.zip) and extract it into a folder called `public` in the root of the project. These assets are based on [this Angular + Spring tutorial](http://websystique.com/springmvc/spring-mvc-4-angularjs-example/).

Add the following to `application.properties`:

```
spring.datasource.url=jdbc:h2:./main
spring.jpa.generate-ddl=true
spring.jpa.hibernate.ddl-auto=none
```

When you run it, you should see a UI in your browser and errors in the JavaScript console. We need to write the appropriate routes to make it work.

The tricky thing about Angular, and other JS frameworks, is that they send query parameters to the server as JSON. Thus far, we've always assumed they were encoded the "old school" way, URL encoding. To demonstrate this, here are two simplified examples of what each type of HTTP request looks like:

The traditional way using URL encoding:
```
POST /blog/posts
Content-Type: application/x-www-form-urlencoded

title=Hello%20World!&body=This%20is%20my%20first%20post!
```

The newer way using JSON encoding:
```
POST /blog/posts
Content-Type: application/json

{"title":"Hello World!","body":"This is my first post!”}
```

To correctly parse this on the server side, we need to stop receiving our query parameters the way we used to.

Let's begin by creating our user entity in `src/main/java/com/theironyard/entities/User.java`. We determined the field names by looking at how the front end names them:

```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue
    int id;

    @Column(nullable = false)
    String username;

    @Column(nullable = false)
    String address;

    @Column(nullable = false)
    String email;
}
```

Don't bother creating a constructor, but do add getters and setters. Now create the repository to go with it in `src/main/java/com/theironyard/services/UserRepository.java`:

```java
public interface UserRepository extends CrudRepository<User, Integer> {
}
```

Finally, create the controller in `src/main/java/com/theironyard/controllers/AngularSpringController.java`. Add the repository and create the `/user` GET route:

```java
@RestController
public class AngularSpringController {
    @Autowired
    UserRepository users;

    @RequestMapping(path = "/user", method = RequestMethod.GET)
    public List<User> getUsers() {
        return (List<User>) users.findAll();
    }
}
```

This should remove one of the errors in the JS console. Next, add the POST route:

```java
@RestController
public class AngularSpringController {
    ...
    
    @RequestMapping(path = "/user", method = RequestMethod.POST)
    public void addUser(@RequestBody User user) {
        users.save(user);
    }
}
```

Notice the `@RequestBody`. It allows us to simply receive all the query paramters at once using our entity's class as the parameter type. Next, add the PUT route. It passes the user's ID in the URL itself, known as a path variable. We can capture that like this:

```java
@RestController
public class AngularSpringController {
    ...
    
    @RequestMapping(path = "/user", method = RequestMethod.PUT)
    public void updateUser(@RequestBody User user) {
        users.save(user);
    }
}
```

Lastly, the routes for deleting and getting an individual user do not need a `@RequestBody` at all, because all they need is the ID. For them, we just use the `@PathVariable` alone:

```java
@RestController
public class AngularSpringController {
    ...
    
    @RequestMapping(path = "/user/{id}", method = RequestMethod.DELETE)
    public void deleteUser(@PathVariable("id") int id) {
        users.delete(id);
    }

    @RequestMapping(path = "/user/{id}", method = RequestMethod.GET)
    public User getUser(@PathVariable("id") int id) {
        return users.findOne(id);
    }
}
```

Testing these routes will be a little different than we've done in the past. First, create the `src/test/resources` directory and mark it as "Test Resources". Then create `src/test/resources/application.properties` with the following:

```
spring.datasource.url=jdbc:h2:mem:test
spring.jpa.generate-ddl=true
```

Edit `AngularSpringApplicationTests.java` to create the `mockMvc` object:

```java
@RunWith(SpringJUnit4ClassRunner.class)
@SpringApplicationConfiguration(classes = AngularSpringApplication.class)
@WebAppConfiguration
public class AngularSpringApplicationTests {

    @Autowired
    WebApplicationContext wap;

    MockMvc mockMvc;
    
    @Before
    public void before() {
        mockMvc = MockMvcBuilders.webAppContextSetup(wap).build();
    }
}
```

We'll start by testing the POST route. To do so, we must create a `User` object and serialize it to JSON. We'll also need to set the content type to "application/json":

```java
@RunWith(SpringJUnit4ClassRunner.class)
@SpringApplicationConfiguration(classes = AngularSpringApplication.class)
@WebAppConfiguration
public class AngularSpringApplicationTests {

	@Autowired
	UserRepository users;

    ...

    @Test
    public void addUser() throws Exception {
        User user = new User();
        user.setUsername("Alice");
        user.setAddress("17 Princess St");
        user.setEmail("alice@gmail.com");

        ObjectMapper mapper = new ObjectMapper();
        String json = mapper.writeValueAsString(user);

        mockMvc.perform(
                MockMvcRequestBuilders.post("/user")
                .content(json)
                .contentType("application/json")
        );

        Assert.assertTrue(users.count() == 1);
    }
}
```

Now let's test the DELETE route:

```java
@RunWith(SpringJUnit4ClassRunner.class)
@SpringApplicationConfiguration(classes = AngularSpringApplication.class)
@WebAppConfiguration
public class AngularSpringApplicationTests {

    ...
    
    @Test
    public void deleteUser() throws Exception {
        mockMvc.perform(
                MockMvcRequestBuilders.delete("/user/1")
        );

        Assert.assertTrue(users.count() == 0);
    }
}
```

## AnonUpload

Create a project via Spring Initializr called `AnonUpload` with the following libraries:

* Web
* DevTools
* JPA
* H2

Let's begin with the client side. Create a `public` folder. Then download [jQuery](http://jquery.com/download/) and move it into it. Then create `public/index.html`:

```html
<html>
<body>
<form action="/upload" enctype="multipart/form-data" method="post">
    <input type="file" name="file"/>
    <button type="submit">Upload</button>
</form>
<div id="fileList"></div>
<script src="jquery-2.2.1.js"></script>
</body>
</html>
```

Now we can work on the server side. Create `src/main/java/com/theironyard/entities/AnonFile.java`:

```java
@Entity
@Table(name = "files")
public class AnonFile {
    @Id
    @GeneratedValue
    int id;

    @Column(nullable = false)
    String filename;

    @Column(nullable = false)
    String originalFilename;

    public AnonFile() {
    }

    public AnonFile(String filename, String originalFilename) {
        this.filename = filename;
        this.originalFilename = originalFilename;
    }
}
```

Create getters and setters for it. Then create your `src/main/java/com/theironyard/services/AnonFileRepository.java`:

```java
public interface AnonFileRepository extends CrudRepository<AnonFile, Integer> {
}
```

Finally, create `src/main/java/com/theironyard/controllers/AnonFileController.java` and write the `/upload` route:

```java
@RestController
public class AnonFileController {
    @Autowired
    AnonFileRepository files;

    @RequestMapping(path = "/upload", method = RequestMethod.POST)
    public void upload(MultipartFile file, HttpServletResponse response) throws IOException {
        File dir = new File("public/files");
        dir.mkdirs();
        File f = File.createTempFile("file", file.getOriginalFilename(), dir);
        FileOutputStream fos = new FileOutputStream(f);
        fos.write(file.getBytes());

        AnonFile anonFile = new AnonFile(f.getName(), file.getOriginalFilename());
        files.save(anonFile);

        response.sendRedirect("/");
    }
}
```

Make sure upload works correctly. Now let's get the file list to display. Create a `/files` route:

```java
@RestController
public class AnonFileController {
    ...
    
    @RequestMapping(path = "/files", method = RequestMethod.GET)
    public List<AnonFile> getFiles() {
        return (List<AnonFile>) files.findAll();
    }
}
```

Then create `public/main.js`:

```js
function getFiles(filesData) {
    for (var i in filesData) {
        var elem = $("<a>");
        elem.attr("href", "files/" + filesData[i].filename);
        elem.text(filesData[i].originalFilename);
        $("#fileList").append(elem);
        var elem2 = $("<br>");
        $("#fileList").append(elem2);
    }
}

$.get("/files", getFiles);
```
