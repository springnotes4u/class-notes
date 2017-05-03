## IronGram

Create a project via Spring Initializr called `IronGram` with the following libraries:

* Web
* DevTools
* JPA
* H2

This time we're going to use H2 exclusively. It tends to work better for group projects, since you can build a JAR file and have other people run it without having to install and set up an external database. Add the following to `application.properties`:

```
spring.datasource.url=jdbc:h2:./main
spring.jpa.generate-ddl=true
spring.jpa.hibernate.ddl-auto=none
```

Start by creating your entities in `src/main/java/com/theironyard/entities/`:

```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue
    int id;

    @Column(nullable = false, unique = true)
    String name;

    @Column(nullable = false)
    String password;

    public User() {
    }

    public User(String name, String password) {
        this.name = name;
        this.password = password;
    }
}
```

```java
@Entity
@Table(name = "photos")
public class Photo {
    @Id
    @GeneratedValue
    int id;

    @ManyToOne
    User sender;

    @ManyToOne
    User recipient;

    @Column(nullable = false)
    String filename;

    public Photo() {
    }

    public Photo(User sender, User recipient, String filename) {
        this.sender = sender;
        this.recipient = recipient;
        this.filename = filename;
    }
}
```

Add getters and setters to both classes, so they can be serialized to JSON properly. Then create your repositories in `src/main/java/com/theironyard/services/`:

```java
public interface UserRepository extends CrudRepository<User, Integer> {
    User findFirstByName(String name);
}
```

```java
public interface PhotoRepository extends CrudRepository<Photo, Integer> {
}
```

Now let's create `src/main/java/com/theironyard/controllers/IronGramController.java` and add the repositories:

```java
@RestController
public class IronGramController {
    @Autowired
    UserRepository users;

    @Autowired
    PhotoRepository photos;
}
```

Now let's set it up to display the H2 web server. First, we need to change the H2 dependency so it is available at compile time. Go into `build.gradle` and in the `dependencies` block, make sure the H2 dependency is brought it with the `compile` function:

```groovy
...

dependencies {
    compile('org.springframework.boot:spring-boot-starter-data-jpa')
    compile('org.springframework.boot:spring-boot-devtools')
    compile('org.springframework.boot:spring-boot-starter-web')
    compile('com.h2database:h2')
    testCompile('org.springframework.boot:spring-boot-starter-test') 
}

...
```

Now we have access to its classes. Let's make it start and stop via the `@PostConstruct` and `@PreDestroy` methods:

```java
@RestController
public class IronGramController {
    ...

    Server dbui = null;

    @PostConstruct
    public void init() throws SQLException {
        dbui = Server.createWebServer().start();
    }

    @PreDestroy
    public void destroy() {
        dbui.stop();
    }
}
```

We are now ready to start our routes. Download [PasswordStorage.java](https://raw.githubusercontent.com/defuse/password-hashing/master/PasswordStorage.java) and move it into the following path in your project: `src/main/java/com/theironyard/utilities/PasswordStorage.java`

```java
@RestController
public class IronGramController {
    ...
    
    @RequestMapping(path = "/login", method = RequestMethod.POST)
    public User login(String username, String password, HttpSession session, HttpServletResponse response) throws Exception {
        User user = users.findByName(username);
        if (user == null) {
            user = new User(username, PasswordStorage.createHash(password));
            users.save(user);
        }
        else if (!PasswordStorage.verifyPassword(password, user.getPassword())) {
            throw new Exception("Wrong password");
        }
        session.setAttribute("username", username);
        response.sendRedirect("/");
        return user;
    }

    @RequestMapping("/logout")
    public void logout(HttpSession session, HttpServletResponse response) throws IOException {
        session.invalidate();
        response.sendRedirect("/");
    }

    @RequestMapping(path = "/user", method = RequestMethod.GET)
    public User getUser(HttpSession session) {
        String username = (String) session.getAttribute("username");
        return users.findByName(username);
    }
}
```

Create a `public` folder. Then download [jQuery](http://jquery.com/download/) and move it into it. Then create `public/index.html`:

```html
<html>
<body>
<form action="/login" method="post" id="login" hidden>
    <input type="text" placeholder="Username" name="username"/>
    <input type="password" placeholder="Password" name="password"/>
    <button type="submit">Login</button>
</form>

<form action="/logout" method="post" id="logout" hidden>
    <button type="submit">Logout</button>
</form>

<script src="jquery-2.1.4.min.js"></script>
</body>
</html>
```

Now we need to write some JavaScript that shows the login form if not logged in, and the logout form if logged in. Create `public/main.js`:

```js
function getUser(userData) {
    if (userData.length == 0) {
        $("#login").show();
    }
    else {
        $("#logout").show();
    }
}

$.get("/user", getUser);
```

Then include `main.js` in your HTML:

```html
...

<script src="jquery-2.1.4.min.js"></script>
<script src="main.js"></script>
</body>
</html>
```

You should now be able to log in and log out. Let's add the upload functionality now. We'll start by adding an upload form in `index.html`:

```html
...

<form action="/upload" method="post" enctype="multipart/form-data" id="upload" hidden>
    <input type="text" placeholder="Receiver name" name="receiver"/>
    <input type="file" name="photo"/>
    <button type="submit">Upload</button>
</form>

<div id="photos"></div>

<script src="jquery-2.1.4.min.js"></script>
<script src="main.js"></script>
</body>
</html>
```

Then add an `/upload` route in the controller:

```java
@RestController
public class IronGramController {
    ...

    @RequestMapping("/upload")
    public Photo upload(
            HttpSession session,
            HttpServletResponse response,
            String receiver,
            MultipartFile photo
    ) throws Exception {
        String username = (String) session.getAttribute("username");
        if (username == null) {
            throw new Exception("Not logged in.");
        }

        User senderUser = users.findFirstByName(username);
        User receiverUser = users.findFirstByName(receiver);

        if (receiverUser == null) {
            throw new Exception("Receiver name doesn't exist.");
        }

        if (!photo.getContentType().startsWith("image")) {
            throw new Exception("Only images are allowed.");
        }

        File photoFile = File.createTempFile("photo", photo.getOriginalFilename(), new File("public"));
        FileOutputStream fos = new FileOutputStream(photoFile);
        fos.write(photo.getBytes());

        Photo p = new Photo();
        p.sender = senderUser;
        p.receiver = receiverUser;
        p.filename = photoFile.getName();
        photos.save(p);

        response.sendRedirect("/");

        return p;
    }
}
```

Next we need to create a GET route to retrieve the photos:

```java
@RestController
public class IronGramController {
    ...
    
    @RequestMapping("/photos")
    public List<Photo> showPhotos(HttpSession session) throws Exception {
        String username = (String) session.getAttribute("username");
        if (username == null) {
            throw new Exception("Not logged in.");
        }

        User user = users.findFirstByName(username);
        return photos.findByReceiver(user);
    }
}
```

Next we need to create a spot in the `index.html` to store the photos we retrieve from that route:

```html
...

<div id="photos"></div>

<script src="jquery-2.1.4.min.js"></script>
<script src="main.js"></script>
</body>
</html>
```

Then we can define a JavaScript function to retrieve the photos from that route:

```js
function getPhotos(photosData) {
    for (var i in photosData) {
        var photo = photosData[i];
        var elem = $("<img>");
        elem.attr("src", photo.filename);
        $("#photos").append(elem);
    }
}

getPhotos();

...
```

## Paging

Open the `Purchases` assignment we did before. We're going to add paging to it. First, change `PurchaseRepository` to extend `PagingAndSortingRepository` instead, and add `Pageable pageable` to the custom method:

```java
public interface PurchaseRepository extends PagingAndSortingRepository<Purchase, Integer> {
    Page<Purchase> findByCategory(Pageable pageable, String category);
}
```

In the controller, add an `Integer page` parameter. It is a boxed integer so it can be optional. 

```java
@Controller
public class PurchasesController {
    ...
    
    @RequestMapping(path = "/", method = RequestMethod.GET)
    public String home(Model model, String category, Integer page) {
        page = (page == null) ? 0 : page;
        PageRequest pr = new PageRequest(page, 10);
        Page<Purchase> p;
        if (category != null) {
            p = purchases.findByCategory(pr, category);
        }
        else {
            p = purchases.findAll(pr);
        }
        model.addAttribute("purchases", p);
        model.addAttribute("nextPage", page+1);
        model.addAttribute("showNext", p.hasNext());
        return "home";
    }
}
```

Then, edit `home.html` to display a "Next" link:

```html
...

{{#showNext}}
<a href="/?page={{nextPage}}">Next</a>
{{/showNext}}
</body>
</html>
```

Now let's make the paging work for the category pages as well. Currently, hitting "Next" after going to a category page will cause it to stop filtering. First, in the controller, pass the category to the template:

```java
@Controller
public class PurchasesController {
    ...
    
    @RequestMapping(path = "/", method = RequestMethod.GET)
    public String home(Model model, String category, Integer page) {
        ...
        
        model.addAttribute("category", category);
        return "home";
    }
}
```

Then edit `home.html` to pass the category into the "Next" link if it isn't null:

```html
...


{{#showNext}}
<a href="/?page={{nextPage}}{{#category}}&category={{.}}{{/category}}">Next</a>
{{/showNext}}
</body>
</html>
```