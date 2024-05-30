# Forum development: Unit 4 Project
# Criteria C
## List of techniques used
### Techinques Used
- Sessions
- Parameterized Queries
- Subqueries
- Dynamic HTML (Jinja2)

## Login system: Tracking user information

My client requires a login system for the webapp so that different users can have unique profile pages and post comments. Initially, I used cookies as a way of storing when a user is logged in, but through my research about cookies and testing it in browser, I found out that the cookie is not a secure way of storing user information because they are stored on the client end.<sup>1</sup>

Instead, I decided to use sessions which are safer according to my research<sup>2</sup> and stored on the server side instead.


``` ['current_user_id'] = db.search(query=f"Select id from users where username='{username}'", multiple=False)```

The user is given a unique id when they log in which is stored on the server, based on the id of the user assigned to them in the users table, and can be accessed from any page or routing in the webapp, through the following function:

```current_user_id = int(session.get('current_user_id')[0])```

## Parameterized Queries

While working, I faced a problem where the queries I wrote were vulnerable to injection attacks, which made them unsafe, so I parameterized them to make them more secure.<sup>3</sup>

```
class DatabaseBridge:
    def run_query(self, query:str, params=None):
        self.cursor.execute(query, params)
        self.connect.commit()
        
    # more functions defined below
```

I decided to use parametirized queries ```params``` because they add a layer of security as it protects my client and their users from injection attacks since the variable being inserted is kept seperate from the SQL query and cannot be manipulated. With parameterized queries I also improve the performance of the code because the database can compile the query once and then execute it multiple times with different paramters. I used object oriented-programming to apply this to every query by defining the function to run queries and using it on any other query later. This is an example of my use of the DRY (don't repeat yourself) paradigm.

## Database Searching Follow Filter

My client required a way to only view posts made by users they follow in their profile and the . I tried storing them all in one table, but many problems arose, including poor data organization, poor performance as the size of the table increased, and anomalies as redundancy replace important data during insertions.

Instead, I am joining multiple tables in a more organized manner upon each other to retrieve different data depending on the multiple factors. This demonstrates my ability to think ahead as I identify what inputs and outputs are needed for the posts function, and my recursive thinking and decomposition skills as I break the problem down into smaller, more repetitive parts (select, join, filter).

```.py
posts_2 = db.search("SELECT posts.*, users.username, users.image, users.email, topics.color "
   "FROM posts JOIN users ON posts.user_id = users.id JOIN topics ON     posts.topic = topics.topic JOIN follows ON posts.user_id = follows.following "
  "WHERE follows.user_id = ? and posts.user_id = follows.following and follows.following IN "
  "(SELECT following FROM follows WHERE user_id = ?) ORDER BY date DESC", multiple=True, params=(user_id, current_user_id))

```

Firstly, I identified all the data needed for a post. That is all the information about the post, as well as the username, image, and email of the post author and the color of the topic the post is under: ```"SELECT posts.*, users.username, users.image, users.email, topics.color "```. In order to get the correct information about the post_author, I **join** the user_id column of the posts table **on** the id column of the users table ```"FROM posts JOIN users ON posts.user_id = users.id"```. I did the same for the topics color, **joining** the topic column in the posts table **on** the topics in the topic tabble, therefore giving its color on the same row ```"JOIN topics ON posts.topic = topics.topic"```. 

Secondly, to filter the table, I checked that the post author is not followed by the current user. From the follows table, I only want to look at users that current user follows (Iknow who the user is through sessions explained above.) ```"WHERE follows.user_id = ?", params=(current_user_id)```. Then I checked that the post author is a user that the current user follows by matching their ids together ```"and posts.user_id = follows.following"```. I also have a subquery filter<sup>4</sup> ```follows.following IN "(SELECT following FROM follows WHERE user_id = ?)``` that pulls all users that the current user follows, and the main query that checks if the users that could be followed are specifically **in** the users provided by the subquery. This may seem unecessary, but I saw through testing that it is important because the follows table includes both users the current user follows and is being followed by, but this search should only inlude posts by the former.

## Follow button

My client required a system that allowed users to follow each other on the webapp so that they can later get posts from only those users. To do that, I implemented a follow button that saved the information of which user followed which users in the follows table. The button would be found on a post, so clicking would mean following the post author. This is convenient as when users realize they are interested in another users posts, they can instantly follow them.

A problem I faced is when creating this button is cases where the current user follows the post author, the webapp still showed a follow button. It would make more sense for it to show an unfollow button instead. Firstly, I tried switching the button functionality depending on the results of the database filter search shown above, but that did not work because the buttons on other posts by the same author had to change when the user followed them, and the data on following information was only queried once on get.

Instead, I solved by adding a following state column in the posts table. The state of this column is decided once on get by comparing to the query from the earlier explanation and then updated **again** everytime the follow button is pressed ```db.run_query(f"UPDATE posts SET following_state = 'Following' WHERE id = ?", (post_id,))```. This solution uses many if statements, first to link to the specific button but then to determine whether to set the states to following or not ex: (```if following_state == 'Not_following':```). It reloads the page everytime, and also affects the follows table for other functions ```db.run_query(f"DELETE FROM follows WHERE user_id = ? AND following = ?", (current_user_id, post_id))```

Another issue I ran into was when the post author was the user currently using the webapp. In that case, I repeated the same process with a new column in posts called myself, but this one housed a boolean value instead. In the case where the post author matched the current user, the follow button would be replaced with a delete button and the page would be reloaded, a function that had to be added anyway ```if myself:db.run_query(f"delete from posts where id = ?", (post_id,))```. It creates a clear difference between what posts are for the user and which ones are not.

```.py
posts_2 = db.search("SELECT posts.*, users.username, users.image, users.email, topics.color FROM posts JOIN users ON posts.user_id = users.id JOIN topics ON posts.topic = topics.topic JOIN follows ON posts.user_id = follows.following WHERE follows.user_id = ? and posts.user_id = follows.following ORDER BY date DESC", multiple=True, params=(user_id,))
    for p in posts_2:
        db.run_query(f"UPDATE posts SET myself = ?", (False,))
    for p in posts_2:
        db.run_query(f"update posts set myself = TRUE where user_id = ?", (current_user_id,))
        if does_user_follow(current_user_id, int(p[1])):
            db.run_query(f"update posts set following_state = 'Unfollow' where user_id = ?", (current_user_id,))
        else:
            db.run_query("UPDATE posts SET following_state = 'Follow' WHERE user_id = ?", (current_user_id,))

if request.form.get('follow'): # here you must program the functionality of each follow button
    # databse queries for checks explained above

    if myself: # this will check if the post author is the current user and switch the follow button to delete if that is true 
        db.run_query(f"delete from posts where id = ?", (post_id,))

    if following_state == 'Not_following': # these queries will follow the post author
        db.insert(f"INSERT INTO follows (user_id, following) VALUES (?, ?)", (current_user_id, post_id))
        db.run_query(f"UPDATE posts SET following_state = 'Following' WHERE id = ?", (post_id,))

    else: # these queries will unfollow the post author
        db.run_query(f"DELETE FROM follows WHERE user_id = ? AND following = ?", (current_user_id, post_id))
        db.run_query(f"UPDATE posts SET following_state = 'Not_following' WHERE id = ?", (post_id,))

```

The last problem I ran into was being unable to display this in HTML templates because they are static. If I wrote down a button as follow, even if its functionailty had changed, it would still show as the follow button. To solve this issue, I used the template editing language Jinja2, because it is made for Python which is what I am using for the back-end development. Jinja2 is dynamic, which means when I render template, I include the post variable as an input for Jinja2, which then uses for loops and if statements to generate multiple posts, and give them different buttons depending on the following state.

```.html
{% for p in posts %}
    <!-- additional design elements and divs -->
    <form method="POST">
      {% if p[8] == "Not_following" %}
                <button name="follow" value="{{ p[1] }}" class="btn--sm uppercase">Follow</button>
            {% elif p[10] == True %}
                <button name="follow" value="{{ p[1] }}" class="btn--sm text-red uppercase">Delete</button>
            {% else %}
                <button name="follow" value="{{ p[1] }}" class="btn--sm uppercase">Unfollow</button>
            {% endif %}
        </form>
{% endfor %}
```

# Appendix
<sup>1</sup>https://www.infosecinstitute.com/resources/general-security/cookies-an-overview-of-associated-privacy-and-security-risks/#:~:text=Yet%2C%20depending%20on%20how%20cookies,user%20and%20gain%20unauthorized%20access.
<sup>2</sup>https://medium.com/@hendelRamzy/how-session-and-cookies-works-640fb3f349d1
<sup>3</sup>https://www.dbvis.com/thetable/parameterized-queries-in-sql-a-guide/#:~:text=Parameterized%20queries%20in%20SQL%20are,attacks%20unfeasible%20for%20the%20attacker.
<sup>4</sup>https://mode.com/sql-tutorial/sql-sub-queries 
