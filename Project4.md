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

## Database Searching Follow Filter
My client required a way to only view posts made by users they follow in their profile. To do this, I am joining multiple tables upon each other to retrieve different data depending on the multiple factors. ```py posts_2 = db.search("SELECT posts.*, users.username, users.image, users.email, topics.color FROM posts JOIN users ON posts.user_id = users.id JOIN topics ON posts.topic = topics.topic JOIN follows ON posts.user_id = follows.following WHERE follows.user_id = ? and posts.user_id = follows.following and follows.following IN (SELECT following FROM follows WHERE user_id = ?) ORDER BY date DESC", multiple=True, params=(user_id, current_user_id))```
