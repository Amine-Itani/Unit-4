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

My client required a system that could save data permanenetly to be used later (posts, username and password...). 

Abstraction and parameterized queries.

## Database Searching Follow Filter
My client required a way to only view posts made by users they follow in their profile. To do this, I am joining multiple tables upon each other to retrieve different data depending on the multiple factors. This demonstrates my ability to think ahead as I identify what inputs and outputs are needed for the posts function, and my recursive thinking and decomposition skills as I break the problem down into smaller, more repetitive parts (select, join, filter).

```.py
posts_2 = db.search("SELECT posts.*, users.username, users.image, users.email, topics.color "
   "FROM posts JOIN users ON posts.user_id = users.id JOIN topics ON     posts.topic = topics.topic JOIN follows ON posts.user_id = follows.following "
  "WHERE follows.user_id = ? and posts.user_id = follows.following and follows.following IN "
  "(SELECT following FROM follows WHERE user_id = ?) ORDER BY date DESC", multiple=True, params=(user_id, current_user_id))

```

Although I could have chosen to use one database and avoid this joining, there are many disadvantages to using only one table including poor data organization, poor performance as the size of the table increases, and anomalies as redundancy replace important data during insertions.

Firstly, it is important to recognize what data is needed for the post. That is all the information about the post, as well as the username, image, and email of the post author and the color of the topic the post is under: ```"SELECT posts.*, users.username, users.image, users.email, topics.color "```. In order to get the correct information about the post_author, we **join** the user_id column of the posts table **on** the id column of the users table ```"FROM posts JOIN users ON posts.user_id = users.id"```. We do the same for the topics color, **joining** the topic column in the posts table **on** the topics in the topic tabble, therefore giving its color on the same row ```"JOIN topics ON posts.topic = topics.topic"```. 

Secondly, to filter the table, we need to check that the post author is not followed by the current user. From the follows table, we only want to look at users that current user follows (we know who the user is through sessions explained above.) ```"WHERE follows.user_id = ?", params=(current_user_id)```. Then we check that the post author is a user that the current user follows by matching their ids together ```"and posts.user_id = follows.following"```. We also have a subquery filter<sup>2</sup> ```follows.following IN "(SELECT following FROM follows WHERE user_id = ?)``` that pulls all users that the current user follows, and the main query that checks if the users that could be followed are specifically **in** the users provided by the subquery. This may seem unecessary, but I saw through testing that it is important because the follows table includes both users the current user follows and is being followed by, but this search should only inlude posts by the former.

Lastly, the posts are order by date descending, as in dates with the smallest value distance of time from the current time ```ORDER BY date DESC```. This is because most recently posted posts tend to be the most relevant one to users.


# Appendix
<sup>2</sup>https://mode.com/sql-tutorial/sql-sub-queries 
