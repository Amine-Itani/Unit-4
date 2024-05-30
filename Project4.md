# Forum development: Unit 4 Project
# Criteria C
## List of techniques used
### Techinques Used
- Server-side Sessions
- Parameterized Queries
- Subqueries
- Dynamic HTML (Jinja2)
- Object Oriented Programming: Abstraction
- Programming Paradigm: DRY (Don't repeat yourself)
- If statements
- Password Validation
- For loops
- Try/except troubleshooting
- In-line CSS

## Login system: Tracking user information

My client requires a login system allowing different users to have profile pages and post comments. Initially, I used cookies to store a user log in, but through my research about cookies and testing it in browser, I realized that cookies are not a secure way of storing user information because they are stored on the client end.<sup>1</sup>

Instead, I decided to use sessions which are safer according to my research<sup>2</sup> because they are stored on the server side .


``` ['current_user_id'] = db.search(query=f"Select id from users where username='{username}'", multiple=False)```

The user is given a unique id when they log in which is stored on the server, based on the id of the user assigned to them in the users table, and can be accessed from any page or routing in the webapp, through the following function:

```current_user_id = int(session.get('current_user_id')[0])```

## Parameterized Queries

While working, I faced a problem where queries I wrote were vulnerable to injection attacks from evil actors, which made them unsafe, so I parameterized them to increase security.<sup>3</sup>

```
class DatabaseBridge:
    def run_query(self, query:str, params=None):
        self.cursor.execute(query, params)
        self.connect.commit()
        
    # more functions defined below
```

I decided to use parametirized queries because they protect users from injection attacks since the variable being inserted is kept seperate from the SQL query and cannot be manipulated. I also improve the performance of the code because the database can compile the query once and then execute it multiple times with different paramters. I used object oriented-programming to apply this to every query by defining the function to run queries in a class and using it on any other query later. This is an example of the DRY (don't repeat yourself) paradigm.

## Database Searching Follow Filter

My client required a way to only view posts made by users they follow. I tried storing them all in one table, but many problems arose, including poor data organization, poor performance as the size of the table increased, and anomalies as redundancy=ies replace important data during insertions.

Instead, I joined multiple tables in an organized manner upon themselves to retrieve different data depending on the multiple factors. This demonstrates my ability to think ahead as I identify what inputs and outputs are needed for post  functions, and my recursive thinking and decomposition skills as I break the problem down into smaller, more repetitive parts (select, join, filter).

```.py
posts_2 = db.search("SELECT posts.*, users.username, users.image, users.email, topics.color "
   "FROM posts JOIN users ON posts.user_id = users.id JOIN topics ON     posts.topic = topics.topic JOIN follows ON posts.user_id = follows.following "
  "WHERE follows.user_id = ? and posts.user_id = follows.following and follows.following IN "
  "(SELECT following FROM follows WHERE user_id = ?) ORDER BY date DESC", multiple=True, params=(user_id, current_user_id))

```

Firstly, I identified the data needed for posts that being the post, the username, image, and email of the post author and the color of its topic: ```"SELECT posts.*, users.username, users.image, users.email, topics.color "```. To get the correct information about the post author, I **join** the user_id column of the posts table **on** the id column of the users table ```"FROM posts JOIN users ON posts.user_id = users.id"```. I did the same for the topics color, **joining** the topic column in the posts table **on** the topics in the topic table, giving its color on the same row ```"JOIN topics ON posts.topic = topics.topic"```. 

Secondly, to filter the table, I checked that the post author is not followed by the current user. From the follows table, I only want to receive users that the current user follows (I know who the user is through sessions explained above.) ```"WHERE follows.user_id = ?", params=(current_user_id)```. Then I checked that the post author is a user that the current user follows by matching their ids together ```"and posts.user_id = follows.following"```. I also have a subquery filter<sup>4</sup> ```follows.following IN "(SELECT following FROM follows WHERE user_id = ?)``` pull all users that the current user follows, and the main query that checks if the users that could be followed are specifically **in** the users provided by the subquery. This may seem unecessary, but I saw through testing that it is important because the follows table includes both users the current user follows and is being followed by, but this search should only inlude posts by the former.

## Follow button

My client required a system allowing users to follow each other on the webapp so they can later get posts from their following. To do that, I implemented a follow button that saves follow links in the follows table. The button would be found on a post, so clicking it would mean following the post author. This is convenient as when users realize they are interested in another users posts, they can instantly follow them.

A problem I faced is creating this button in cases where the current user already follows the post author, the webapp showed a follow button instead of an unfollow. Firstly, I tried switching the button functionality depending on the results of the database filter search shown above, but that did not work because the buttons on other posts by the same author did not change when the user followed them, and the data on following information was queried once on loading the page.

Instead, I added a following state column in the posts table. This column is decided once on get by comparing to the query from the earlier explanation and **updated** everytime the follow button is pressed ```db.run_query(f"UPDATE posts SET following_state = 'Following' WHERE id = ?", (post_id,))```. This solution uses many if statements, to link to the specific button and to determine whether to set states to following or not ex: (```if following_state == 'Not_following':```). It reloads the page everytime to show results, and affects the follows table ```db.run_query(f"DELETE FROM follows WHERE user_id = ? AND following = ?", (current_user_id, post_id))```

Another issue I found was when the post author was the user currently using the webapp. In that case, I repeated the same process with a new column in posts called myself, but this housed a boolean value instead. In the case where the post author matched the current user, the follow button would become delete and reload the page, adding that function ```if myself:db.run_query(f"delete from posts where id = ?", (post_id,))```.

```.py
posts_2 = db.search("SELECT posts.*, users.username, users.image, users.email, topics.color FROM posts JOIN users ON posts.user_id = users.id JOIN topics ON posts.topic = topics.topic JOIN follows ON posts.user_id = follows.following WHERE follows.user_id = ? and posts.user_id = follows.following ORDER BY date DESC", multiple=True, params=(user_id,))
    for p in posts:
        db.run_query(f"UPDATE posts SET myself = ?", (False,))
    for p in posts:
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

Another problem was the inability to display this in statics HTML. The follow button would display follow even if its functionailty had changed. To solve this, I used the template editing language Jinja2<sup>5</sup>, because it is made for Python which I am using for the back-end development. Jinja2 is dynamic, which means when I render a template, I include the post variable as an input with Jinja's for loops and if statements to generate posts, and give them different buttons depending on the following state.

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

# Criteria D: Video

https://drive.google.com/drive/folders/1qWRrioOV3q5zWRDajPETrpvH3EZE-v_o?usp=drive_link 

<sub>Fig.1 A 3 minute 39 second video demonstrating the functionalities of the webapp</sub>

Due to storage, editing, and organizational reasons, the video had to be split in two.

https://drive.google.com/drive/folders/1lw2t1D6YcUxSXDQh0pxaDxSSE25b_tYS?usp=sharing

<sub>Fig.2 A 25 second video demonstrating the ease another programmer would have expanding this project due to good coding practices</sub>

# Criteria E: Evaluation

A blackbox test was conducted with the help of two, non-compter scietist individuals. With the only given instruction being an explanation on the prupose of the program " It is for a mathematics forum that allows students to work with each other online.", they filled out a form with the pros and cons they faced when using the webapp which can be found in the appendix.<sup>6</sup>

## Meeting Success Criteria

|                    **Success Criteria**                   | **Met?** |                                        **Description**                                       |
|:---------------------------------------------------------:|:--------:|:--------------------------------------------------------------------------------------------:|
|            Hashed login and registration system           |    YES   |   Registration system is easy to use and secure thanks to hashing and password confirmation  |
|            Posting system to EDIT/CREATE/DELETE           |    YES   |                        Comments work well and so does posting system                         |
|                 System to add/remove likes                |    YES   |        Likes system works thanks to counter and inability to like the same post twice        |
| A system to follow/unfollow users, follow/unfollow topics |    YES   | Webapp allows following and unfollowing users, and redirects their posts to the profile page |
|           Profile page with relevant information          |    YES   |     Is functional and the content is that which is followed which is relevant information    |
|                       Upload Images                       |    YES   |               Upload images is possible both for posts and for profile pictures              |
|                        Send emails                        |    YES   |                        Reaching out to users is possible and adequate                        |

## Extensibility

The testers highlighted a few potential improvements in the future of the webapp:
- For quality of life, the description and profile picture of the user can be displayed in their profile. This is an easy fix via some Jinja2 additions in the HTML template
- The follow/unfollow button starts as unfollow although the defualt is unfollow, display does not match on first click only, and that can be improved. This could be possible through a refining of the if statements in HTML through Jinja linked to the display of the button.

# Appendix

## MLA9 Citation Format

<sup>1</sup>Dodt, Claudio. “Cookies: An Overview of Associated Privacy and Security Risks.” Infosec, 7 July 2020, <sub>www.infosecinstitute.com/resources/general-security/cookies-an-overview-of-associated-privacy-and-security-risks/#:~:text=Yet%2C%20depending%20on%20how%20cookies,user%20and%20gain%20unauthorized%20access. </sub>

<sup>2</sup>Ramzy, HENDEL. “How Session and Cookies Works ?” Medium, Medium, 16 Jan. 2024, <sub>medium.com/@hendelRamzy/how-session-and-cookies-works-640fb3f349d1. </sub>

<sup>3</sup>Vileikis, Lukas. “Parameterized Queries in SQL – A Guide.” DbVisualizer, DbVis Software AB, 5 Apr. 2024, <sub>www.dbvis.com/thetable/parameterized-queries-in-sql-a-guide/#:~:text=Parameterized%20queries%20in%20SQL%20are,attacks%20unfeasible%20for%20the%20attacker. </sub>

<sup>4</sup>SQL, Mode. “Writing Subqueries in SQL: Advanced SQL - Mode.” Mode Resources, 23 May 2016, <sub>mode.com/sql-tutorial/sql-sub-queries. </sub>

<sup>5</sup>Ronacher, Armin. “Jinja¶.” Jinja, 5 May 2024, <sub>jinja.palletsprojects.com/en/3.1.x/. </sub>

<sup>6</sup>Primary Source: <sub>Evalutation form transcript: https://docs.google.com/document/d/1rS3_d1NuW42JYIMBdETM351HhKtvDIikywUYfc-6pJI/edit?usp=sharing</sub>
