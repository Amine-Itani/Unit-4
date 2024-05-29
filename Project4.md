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
My client required a way to only view posts made by users they follow in their profile. To do this, I am joining multiple tables upon each other to retrieve different data depending on the multiple factors. Firstly, it is important to recognize what data is needed for the post. That is all the information about the post, as well as the username, image, and email of the post author and the color of the topic the post is under 

```.py SELECT posts.*, users.username, users.image, users.email, topics.color ```
