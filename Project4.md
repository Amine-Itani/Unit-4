# Forum development: Unit 4 Project
# Criteria C
## List of techniques used
### Existing Tools
- Python 3.10.11
- Sqlite 3.39.3
- Jinja2 3.1.4
- HTML 5

### Techinques Used
- Sessions
- Database Interaction

### Tracking user information
Not token-based, not cookies, but sessions. How? And why?

### Database Bridge

My client required a system that could save data permanenetly to be used later (posts, username and password...). To do this, I decided to use sqlite because it is scalable as the client expands the amount of users on the forum and it has quick querying time when compared to other programming langugages used for database management (ex: mongodb). It is also has built-in security features I will cover later.

To access sql queries in the most time-efficient manner for the benefit of the user, I chose to define a Database Bridge class in another file and import its functions to use on the forum database. This is a form of object oriendted programming and abstraction, which is beneficial because it keeps functions for interacting with the database consistent, meaning the user faces less errors in the website in the long-run.

```
class DatabaseBridge:
    def __init__(self,name):
        self.db_name = name
        self.connect = sqlite3.connect(self.db_name)
        self.cursor = self.connect.cursor()

    def run_query(self, query:str, params=None):
        self.cursor.execute(query, params)
        self.connect.commit()
        
    # more functions defined below
```
First, I use the constructor method ```def __init__(self,name):``` to connect to the database ```name``` and create a cursor to execute commands. Beyond that, functions such as the run_query function take in self to reference the instance of the class to execute on the database the constructor connected to. I then pass in a query (ex: ```db.run_query(f"UPDATE posts SET liked_state = 'Liked' WHERE id = ?", (post_id,))```) that the cursor executes and commits to the database. 

I also used parametirized queries<sup>1</sup> ```params``` because they add a layer of security as it protects my client and their users from injection attacks since the variable being inserted is kept seperate from the SQL query and cannot be manipulated. With parameterized queries I also improve the performance of the code because the database can compile the query once and then execute it multiple times with different paramters.

## Database Searching Follow Filter
My client required a way to only view posts made by users they follow in their profile. To do this, I am joining multiple tables upon each other to retrieve different data depending on the multiple factors. This demonstrates my ability to think ahead as I identify what inputs and outputs are needed for the posts function, and my recursive thinking and decomposition skills as I break the problem down into smaller, more repetitive parts (select, join, filter).

```.py
posts_2 = db.search("SELECT posts.*, users.username, users.image, users.email, topics.color "
   "FROM posts JOIN users ON posts.user_id = users.id JOIN topics ON     posts.topic = topics.topic JOIN follows ON posts.user_id = follows.following "
  "WHERE follows.user_id = ? and posts.user_id = follows.following and follows.following IN "
  "(SELECT following FROM follows WHERE user_id = ?) ORDER BY date DESC", multiple=True, params=(user_id, current_user_id))

```

Although I could have chosen to use one database and avoid this joining, there are many disadvantages to using only one table including poor data organization, poor performance as the size of the table increases, and anomalies as redundancy replace important data during insertions.

Firstly, I recognized that I need to acquire all the data needed for a post. That is all the information about the post, as well as the username, image, and email of the post author and the color of the topic the post is under: ```"SELECT posts.*, users.username, users.image, users.email, topics.color "```. In order to get the correct information about the post_author, we **join** the user_id column of the posts table **on** the id column of the users table ```"FROM posts JOIN users ON posts.user_id = users.id"```. I did the same for the topics color, **joining** the topic column in the posts table **on** the topics in the topic tabble, therefore giving its color on the same row ```"JOIN topics ON posts.topic = topics.topic"```. 

Secondly, to filter the table, I checked that the post author is not followed by the current user. From the follows table, we only want to look at users that current user follows (we know who the user is through sessions explained above.) ```"WHERE follows.user_id = ?", params=(current_user_id)```. Then we check that the post author is a user that the current user follows by matching their ids together ```"and posts.user_id = follows.following"```. We also have a subquery filter<sup>2</sup> ```follows.following IN "(SELECT following FROM follows WHERE user_id = ?)``` that pulls all users that the current user follows, and the main query that checks if the users that could be followed are specifically **in** the users provided by the subquery. This may seem unecessary, but I saw through testing that it is important because the follows table includes both users the current user follows and is being followed by, but this search should only inlude posts by the former.

Lastly, the posts are order by date descending, as in dates with the smallest value distance of time from the current time ```ORDER BY date DESC```. This is because most recently posted posts tend to be the most relevant one to users.


# Appendix
<sup>1</sup>https://www.dbvis.com/thetable/parameterized-queries-in-sql-a-guide/#:~:text=Parameterized%20queries%20in%20SQL%20are,attacks%20unfeasible%20for%20the%20attacker.
<sup>2</sup>https://mode.com/sql-tutorial/sql-sub-queries 
