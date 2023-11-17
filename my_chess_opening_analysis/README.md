---

Python Meets Pawn: Decoding My Chess Openings with Data Analysis

In this blog, I'll guide you through the process of analyzing your chess games played on the Chess.com platform using Python.
Chess has always been a passion of mine, a beautiful game introduced to me by my father. My early years were spent playing chess with family, which later transitioned to the digital boards of Chess.com. Recently, there's been a resurgence in chess's popularity, fueled by well-known streamers and the educational efforts of Chess Grandmasters. This new wave of interest sparked a question in my mind during a series on chess openings: 'What are the openings I frequently use, and how successful are they for me?' Realizing I had no insight into my own preferences or success rates, I decided to marry two of my greatest loves: Chess and Python.
Let's get started on understanding the steps, learn how to use the Chess.com API, and find out how to check out your own opening moves in Chess! 
P.S. This blog assumes that Python and preferably Jupyter Notebook (or any other IDE) is already installed on your laptop.

---

Chess.com API
First, you need to install the Chess.com library to use its API. You can do it using the "pip" command in Terminal (or Command Prompt), as well as inside Jupyter Notebook using "!" before the syntax.
pip install chess.com
You can find all the instructions and details at https://chesscom.readthedocs.io/en/latest/. It contains every method and parameter that can be used.
You also need traditional pandas and numpy libraries, which you can install the same way as above.

---

Get the data
First off, let's get all the libraries we need set up, and then make our first request to the API. We'll use a method called 'get_player_games_by_month' to see all the games played in a particular year and month. To get a feel for the type of data we get, we'll look at a sample game. By using Python's built-in 'pprint' library, we can make the JSON response easier to read. 
# Import necessary libraries
from chessdotcom import get_player_game_archives, get_player_games_by_month, Client
import pandas as pd
import numpy as np
from pprint import pprint

# Configure the user agent for the API requests to Chess.com
# this part is mandatory as per new version of API
Client.request_config["headers"]["User-Agent"] = (
   "My Python Application. "
   "Contact me at xxxx@gmail.com"
)

# get games for the month of November 2023
response_sample = get_player_games_by_month("mikayil94", year=2023, month=11)

# print the JSON
pprint(response_sample.json)
The really cool part is in the PGN (Portable Game Notation) section - it has everything we need, like the name of the opening and a link to more details (ECOUrl)
There's a method called 'get_player_game_archives' which helps us get a list of our old games on this platform, sorted by the year and month we played them. The dates come in link format, so we just need to pick out the date part from each link.
# Retrieve a list of months during which the player 'mikayil94' has played games
response1 = get_player_game_archives("mikayil94")
list_of_played_months = []
for i in response1.json['archives']:
    list_of_played_months.append(i[-7:])
Now for the main part! We can use the year and month we found earlier and pass values to the 'get_player_games_by_month' method to get more info about our games. The following columns will be derived from each game: 'time_class', 'date', 'white', 'black', 'game_link', 'opening_code', 'opening_name', 'opening_link', 'result'. The 'time_class' part comes from a different place than the rest, which is all inside the PGN section. What we really need for our analysis are the names of the players (both White and Black), and the name of the opening move. It's also really useful to have a link for each opening move. This way, we can learn more about it and get better at using it. Plus, having the link to the game itself is great because it lets us look back and understand how we won or lost each game.
# Create a DataFrame to store game information
my_games_df = pd.DataFrame(columns = ['time_class', 'date', 'white', 'black', 'game_link', 'opening_code', 'opening_name', 'opening_link', 'result'])

# Loop through each month and retrieve games played in that month
for months in list_of_played_months:
    response2 = get_player_games_by_month("mikayil94", year=months.split("/")[0], month=months.split("/")[1])  
    
    # Extract relevant information from each game and add it to the DataFrame
    for i in response2.json['games']:
        time_class = i['time_class']
        pgn = i['pgn']
        if "ECOUrl" not in pgn : continue  # Skip the game if it doesn't have an ECO URL

        # Extract various details from the PGN (Portable Game Notation) of the chess game
        date = pgn[pgn.find("Date"):].split(" ")[1].split("]")[0].strip('\"')
        white = pgn[pgn.find("White"):].split(" ")[1].split("]")[0].strip('\"')
        black = pgn[pgn.find("Black"):].split(" ")[1].split("]")[0].strip('\"')
        game_link = pgn[pgn.find("Link"):].split(" ")[1].split("]")[0].strip('\"')
        opening_code = pgn[pgn.find("ECO"):].split(" ")[1].split("]")[0].strip('\"')
        opening_name = pgn[pgn.find("ECOUrl"):].split(" ")[1].split("]")[0].split("/")[-1].strip('\"')    
        opening_link = pgn[pgn.find("ECOUrl"):].split(" ")[1].split("]")[0].strip('\"')    
        result = np.where(pgn[pgn.find("Termination"):].split(" ")[1].split("]")[0].strip('\"') == 'mikayil94', 'Win', 'Loss') # if my username is in this field, it means I was the Winner.

        # Create a new DataFrame for the current game and append it to the main DataFrame
        my_games_df_new = pd.DataFrame({'time_class' : [time_class], 'date' : [date], 'white' : [white], 'black' : [black], \
                        'game_link' : game_link, 'opening_code' : opening_code, 'opening_name' : [opening_name], 'opening_link' : [opening_link], 'result' : [result]})
        my_games_df = pd.concat([my_games_df, my_games_df_new], ignore_index=True)   

---

Create variables for the final result
Now that we have our data, we need to add a few things to make it clearer and easier to see what's going on. It's important to know who played the opening move in each game. Did I play against that opening when I was Black or did I use it when I was White? To figure this out, I'll check which side I was on for each game. Then, by looking at whether I won or lost each game, I can work out my win rate for each type of opening.
# Add a new column 'opening_side' to the DataFrame. If the player 'mikayil94' is white, set the value to 'white', otherwise 'black'
my_games_df['opening_side'] = np.where(my_games_df.white == 'mikayil94', 'white', 'black')

# Add a new column 'result_binary'. If the result of the game is 'Win', set the value to 1, otherwise 0
my_games_df['result_binary'] = np.where(my_games_df.result == 'Win', 1, 0)

# Group the DataFrame by opening name, link, code, and the side 'mikayil94' played.
# Aggregate the data to count the total number of wins and total games played for each group
my_openings = my_games_df.groupby(["opening_name", "opening_link", "opening_code", "opening_side"], as_index=False).agg(
    games_win = ('result_binary', 'sum'),  # Sum of 'result_binary' to get total wins
    games_count = ('result_binary', 'count')  # Count of 'result_binary' to get total games played
)

# Calculate the win percentage for each opening and add it as a new column 'win_percentage'
# The win percentage is rounded to two decimal places
my_openings['win_percentage'] = round(my_openings.games_win / my_openings.games_count, 2)

---

And the result is here!
Now we get to see the results! I used matplotlib and seaborn libraries (install using pip if not present) to visualize the data. I created a new variable named "opening_and_side", to be used in visualization, to indicate which side (White or Black) played the opening. I only looked at the openings that I played at least 10 times to make sure my analysis was good.
import matplotlib.pyplot as plt
import seaborn as sns

# Prepare the data for visualization
# Add new column, to concatenate opening name and opening side, which will be used in visualization
my_openings['opening_and_side'] = my_openings.opening_name + '[as ' + my_openings.opening_side + ']'
# filter data to show only games with at least 10 count
viz_data = my_openings[my_openings.games_count > 10].sort_values("win_percentage", ascending=False)[['opening_and_side', 'win_percentage']]

# Create a bar plot
plt.figure(figsize=(15, 10))
sns.barplot(x='win_percentage', y='opening_and_side', data=viz_data, palette="viridis", ci=None)
plt.title('Win Percentage by Chess Opening')
plt.xlabel('Win Percentage')
plt.ylabel('Opening Name')
plt.xticks(rotation=45)
plt.tight_layout()

# Display the plot
plt.show()

---

Key takeaways for me after this analysis:
Owen's Defense! This was my go-to opening in 2018 and 2019, but I never realized I was actually doing well with it until now. It's not a common opening you'd hear streamers talk about, which is great for catching your opponent off guard! Turns out, this opening is pretty solid if you look at games played by the Grandmasters. Black wins 46.3% of the time, while White has a 34.6% win rate. You can see more about it at the Chess Openings Database which can be found here: https://old.chesstempo.com/chess-openings.html.

Good with Barnes-Opening-1…d5–2.e4. I did not know this opening was called Barnes Opening and did not know I had a good winning percentage with it. Even the Chess Openings Database says this is not the best opening for Whites, where they get -0.4 eval after playing f3, which weakens King's side. But since it's not a common opening, it seems to surprise my opponents. In this position, Black should not take the pawn, but my opponents in most cases took it, and that made the game more even.

Vant-Kruijs-Opening - I was always getting into a worse position after playing this opening as White, and also gaining an advantage when the opponent was playing it, so, bad opening! The Chess Openings Database backs this up: it shows that White players only win about 36.5% of the time with this opening, while their opponents win 45.3% of the time! 

Was not performing well while playing against the Kings-Pawn-Opening-Wayward-Queen-Attack. Do not have a recent history with this opening after 2019, where I usually blundered easily, falling into traps, but hey, you live and learn! :)
I haven't done well with the Kings-Pawn-Opening-Napoleon-Attack. Good thing I haven't played it in a long time! Bringing out the Queen so early in the game usually isn't a good move :)
When I play against the Queens-Pawn-Opening-Accelerated-London-System as black, it doesn't usually put me in a bad spot right away. But, looking back, I don't win as much as I'd like with this opening. It seems like I need to spend some time studying and practicing it more.

---

Conclusion
It's really cool that Chess.com has this public API for us to do this kind of fun analysis and find interesting things. From looking at my games, I found out that I actually played better before I started learning about all the famous openings. Sometimes, playing an unusual opening can be a good thing. So, why not surprise your opponents with the Barnes Opening or Owen's Defense? Just be careful not to make any blunders when your opponent plays the Wayward Queen Attack.
Thanks for sticking with me to the end of this! I hope you had fun reading it and that it maybe got you interested in chess, Python, or using Python to look at your own chess games :)
