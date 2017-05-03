## GameTrackerSpring

Sometimes you'll want to query just a subset of your data. Let's add a way to view all the games marked with a specific genre. We'll need to add links in `home.html` that pass the genre as a query parameter:

```html
<html>
<body>
<a href="/">All</a>
<a href="/?genre=adventure">Adventure</a>
<a href="/?genre=rpg">RPG</a>
<a href="/?genre=strategy">Strategy</a>
<a href="/?genre=shooter">Shooter</a>

<br><br>

...
```

Then we need to create a special method that querys by genre. To do so, we must go to `GameRepository` and add the following method signature:

```java
public interface GameRepository extends CrudRepository<Game, Integer> {
    List<Game> findByGenre(String genre);
}
```

Note that the name of this method is sematically meaningful. Any method called `findBySomething(String something)` will produce a SQL query that filters the records by a column named `something`.

Now we must change the `/` route to accept a `genre` query parameter and use the custom query method if it isn't null:

```java
@Controller
public class GameTrackerController {
    ...
    
    @RequestMapping(path = "/", method = RequestMethod.GET)
    public String home(Model model, String genre) {
        List<Game> gameList;
        if (genre != null) {
            gameList = games.findByGenre(genre);
        } else {
            gameList = games.findAll();
        }
        model.addAttribute("games", gameList);
        return "home";
    }
    
    ...
}
```

The links we added to the main page should now work. Just for fun, let's add a filter for release year. Add `findByReleaseYear` to your `GameRepository`:

```java
public interface GameRepository extends CrudRepository<Game, Integer> {
    List<Game> findByGenre(String genre);
    List<Game> findByReleaseYear(int year);
}
```

Then add the necessary parameter to the `/` route. We need to use a boxed integer (`Integer`) rather than the primitive version (`int`). This is because the boxed version can be set to null, whereas the primitive version must have a value set. We want this parameter to be optional, and the only way to make it so is to allow it to be set to null:

```java
@Controller
public class GameTrackerController {
    ...
    
    @RequestMapping(path = "/", method = RequestMethod.GET)
    public String home(Model model, String genre, Integer releaseYear) {
        List<Game> gameList;
        if (genre != null) {
            gameList = games.findByGenre(genre);
        } else if (releaseYear != null) {
            gameList = games.findByReleaseYear(releaseYear);
        } else {
            gameList = games.findAll();
        }
        model.addAttribute("games", gameList);
        return "home";
    }
    
    ...
}
```

You can test that out by adding a game with a given release year, like `1995`, and then going to `http://localhost:8080/?releaseYear=1995`.

There are many endless ways to write query methods. Reference [the documentation](http://docs.spring.io/spring-data/jpa/docs/current/reference/html/#jpa.query-methods.query-creation) to learn the rules. Here is an example of the possibilities:

```java
public interface GameRepository extends CrudRepository<Game, Integer> {
    List<Game> findByUser(User user);
    List<Game> findByReleaseYear(int year);
    List<Game> findByGenreAndReleaseYear(String genre, int releaseYear);
    List<Game> findByReleaseYearIsGreaterThanEqual(int minReleaseYear);

    Game findFirstByGenre(String genre);
    int countByGenre(String genre);
    List<Game> findByGenreOrderByNameAsc(String genre);
}
```

Let's add a search feature. To begin, we'll add the necessary form to the HTML page:

```html
...

<form action="/" method="get">
    <input type="text" placeholder="Search" name="search"/>
    <button type="submit">Search</button>
</form>

<br><br>

...
```

To add a proper search feature, we need to use the SQL operator known as `LIKE` instead of `=`. To do this, we'll need to write a raw SQL query instead of a normal query method like before. Luckily, we can do so using the `@Query` annotation.

```java
public interface GameRepository extends CrudRepository<Game, Integer> {
    ...
    
    @Query("SELECT g FROM Game g WHERE g.name LIKE ?1%")
    List<Game> findByNameStartsWith(String name);
}
```

Note that it is fairly strict about how the query is written. We need to write the class name instead of the table name (thus, `Game` instead of `games`) and provide an alias (that's what the `g` is). We have to write the first argument as `?1`, and `%` as the wildcard character.

Now let's add a way to log into the app and only show the games the current user submitted. To begin, create a `User` class:

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
}
```

Note that we added `unique = true` to the username just as an additional constraint. Also keep in mind that the `name = "users"` bit is important. By default, Hibernate uses the class name as the table name, which would make the table name `user`. This is actually a special keyword in PostgreSQL, so you'll get a cryptic error message if you don't rename it to `users` like we did.

Next, add login functionality to `home.html`:

```java
<html>
<body>
{{^user}}
<form action="/login" method="post">
    <input type="text" placeholder="Username" name="userName" required/>
    <input type="password" placeholder="Password" name="password" required/>
    <button type="submit">Login</button>
</form>
{{/user}}

{{#user}}
Welcome, {{name}}!
<form action="/logout" method="post">
    <button type="submit">Logout</button>
</form>
<br><br>

...

{{/user}}
</body>
</html>
```

We will now need an interface called `UserRepository`:

```java
public interface UserRepository extends CrudRepository<User, Integer> {
    User findFirstByName(String userName);
}
```

We added a custom query method so we can try querying a user by name when they first attempt to login. Now bring the repository into `GameTrackerSpringController` and make the `/login` and `/logout` routes:

```java
@Controller
public class GameTrackerController {
    ...

    @Autowired
    UserRepository users;
    
    @RequestMapping(path = "/login", method = RequestMethod.POST)
    public String login(HttpSession session, String userName, String password) throws Exception {
        User user = users.findFirstByName(userName);
        if (user == null) {
            user = new User(userName, password);
            users.save(user);
        }
        else if (!password.equals(user.getPassword())) {
            throw new Exception("Incorrect password");
        }
        session.setAttribute("userName", userName);
        return "redirect:/";
    }
    
    @RequestMapping(path = "/logout", method = RequestMethod.POST)
    public String logout(HttpSession session) {
        session.invalidate();
        return "redirect:/";
    }
}
```

Now let's link the user to the game they insert. First, add a `user` field in the `Game` entity and adjust the constructor to set it:

```java
@Entity
public class Game {
    ...

    @ManyToOne
    User user;
    
    public Game() {
    }

    public Game(String name, String platform, String genre, int releaseYear, User user) {
        this.name = name;
        this.platform = platform;
        this.genre = genre;
        this.releaseYear = releaseYear;
        this.user = user;
    }
}
```

The `@ManyToOne` annotation causes this table to add a foreign key in the database called `user_id` for the purposes of doing joins. Now edit the `/` route to pass the username to the template:

```java
@Controller
public class GameTrackerController {
    ...

    @RequestMapping(path = "/", method = RequestMethod.GET)
    public String home(HttpSession session, Model model, String genre, Integer releaseYear, String platform) {
        String userName = (String) session.getAttribute("userName");
        User user = users.findFirstByName(userName);
        if (user != null) {
            model.addAttribute("user", user);
        }
        
        ...
    }
    
    ...
}
```

Then change the `/add-game` route to include the user in the `Game` object that you save:

```java
@Controller
public class GameTrackerController {
    ...
    
    @RequestMapping(path = "/add-game", method = RequestMethod.POST)
    public String addGame(HttpSession session, String gameName, String gamePlatform, String gameGenre, int gameYear) {
        String userName = (String) session.getAttribute("userName");
        User user = users.findFirstByName(userName);
        Game game = new Game(gameName, gamePlatform, gameGenre, gameYear, user);
        games.save(game);
        return "redirect:/";
    }
    
    ...
}
```

After trying it out, you can look at the database with `psql` to confirm that the item was added to the `games` table and the `user_id` column has a number in it that points to the correct record in the `users` table.

Lastly, let's learn how to add a user to the database right when the app starts up. To do so, we can make a method with the `@PostConstruct` annotation. Inside of it, we can check if the `users` table has anything in it, and if not, insert a default user:

```java
@Controller
public class GameTrackerController {
    ...
    
    @PostConstruct
    public void init() {
        if (users.count() == 0) {
            User user = new User();
            user.name = "Zach";
            user.password = "hunter2";
            users.save(user);
        }
    }
    
    ...
}
```
