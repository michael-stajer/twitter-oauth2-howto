# Twitter OAuth2 Implementation Guide
How to implement Twitter's OAuth2 (Python)
(the writeup I wish existed before I tried to implement it)

## Twitter developer portal setup
1. Goto https://developer.twitter.com/en/portal
1. create a new project
1. get a twitter client id
1. get a twitter client secret
1. Setup the User Authentication Settings 
-- be sure to setup your local and production callback urls (ie: http://127.0.0.1:8080/callback and https://production.com/callback)

## You are now ready to start the user flow
Example user flow:
1. Present user with a 'authenticate with twitter' link
1. user clicks the link which opens twitter's oauth2 approval screen
1. user logs in to the twitter screen
1. twitter redirects the user back to your /callback url
1. you save the authentication token and start interacting with the user's account

## step 1: generate the 'auth with twitter' link
* start by making a random string that will only be used for this one request

``
    rand_string = "".join(random.choices(string.ascii_uppercase + string.ascii_lowercase + string.digits, k=32))
``

* then, build the url
```    
    url = f'https://twitter.com/i/oauth2/authorize' + \
                '?response_type=code' + \
                f'&client_id={twitter_client_id}' + \
                f'&redirect_uri={callback_uri}&scope=tweet.read+users.read' + \
                f'&state={session_id}' + \
                f'&code_challenge={rand_string}'+ \
                '&code_challenge_method=plain'
 ```
 * session_id: you need to track your user so you know who it is when they are redirected to your /callback
 * twitter_client_id: the client id from twitter
 * callback_uri: where you want twitter to send the user after they login. ** see note about CALLBACK_URI below **
 
 ### A note about CALLBACK_URI
 Twitter wants are urlencoded callback uri. So, don't submit: https://127.0.0.1/callback.
 Do submit: ``https%3A%2F%2F127.0.0.1%3A8080%2Fcallback``
 
 ## step 2: on /callback you need to parse the response then submit to get the access_token
 Using variables from the /callback url, send a request to twitter to get the access_token
 
 ```
    # parse variables from the url
    r = {}
    r['access_code'] = request.args.get('code')
    r['response_url'] = request.url

    # contruct the basic auth header client_id:client_secret base64 encoded
    message = twitter_client_id+":"+twitter_client_secret
    message_bytes = message.encode('ascii')
    base64_bytes = base64.b64encode(message_bytes)
    base64_message = base64_bytes.decode('ascii')
    r['base64_message'] = base64_message


    # Get the access token from twitter
    my_header = {}
    my_header['Content-Type'] =  'application/x-www-form-urlencoded'
    my_header['Authorization'] = 'Basic ' + base64_message

    my_data = {}
    my_data['code'] = r['access_code']
    my_data['grant_type'] = 'authorization_code'
    my_data['redirect_uri'] = 'http://127.0.0.1:8080/callback' 
    my_data['code_verifier'] = session['twitter_code_challenge']
    my_data['code_challenge_method'] = session['twitter_code_challenge_method']


    url = 'https://api.twitter.com/2/oauth2/token'
    x = requests.post(url, headers=my_header, data=my_data)

    r['post_return_text'] = x.json()
    r['access_token'] = r['post_return_text']['access_token']
```

** Save the access_token for the user **

### step 3: You have credentials for the user, do something useful
Let's get some info about the user


```
    # get info about the user
    url = "https://api.twitter.com/2/users/me?"  + \      
        "user.fields=created_at,description,entities,id,location,name,pinned_tweet_id,profile_image_url,protected,public_metrics,url,username,verified,withheld"
    my_header = {}
    my_header["Authorization"] = "Bearer " + r['access_token']
    x = requests.get(url, headers=my_header)
    r['twitter_user_detail'] = x.json()

    this_twitter_user_id = r['twitter_user_detail']['data']['id']
    r['this_twitter_user'] = this_twitter_user_id
    r['twitter_username'] = r['twitter_user_detail']['data']['username']
    r['twitter_follower_count'] = r['twitter_user_detail']['data']['public_metrics']['followers_count']
    r['twitter_following_count'] = r['twitter_user_detail']['data']['public_metrics']['following_count']
    r['twitter_tweet_count'] = r['twitter_user_detail']['data']['public_metrics']['tweet_count']
```

