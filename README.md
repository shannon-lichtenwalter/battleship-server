# Battleship Server

Live app: https://battleship-capstone.now.sh/

Client GitHub: https://github.com/shannon-lichtenwalter/battleship-client

Server GitHub: https://github.com/shannon-lichtenwalter/battleship-server

## API
_________
Unprotected-Endpoints
_____________________

`POST /api/signup` Creates new user if sucessful in signup attempt. Requires body to be an object with 'username' and 'password' keys. Returns 201 if sucessful.

- Username may not be more than 20 characters. 

- Password must be between 8-32 characters and have an Upper/Lower/Number.

- Neither may contain a space. 


`POST /api/login` Logs in a user if supplied with correct information. Requires body to be an object with 'username' and 'password' keys. Returns 200 with JWT if sucessful. 

_____

Protected-Endpoints
___________________


`POST /api/ships` Allows a user to store their ships in the database. Returns 201 if successful. Requires body to be an object in the shape of: 

    [ 
        { 
            name: 'aircraftCarrier',
            length: 5,
            spaces: [ 'A1', 'A2', 'A3', 'A4', 'A5' ] 
        },
        { 
            name: 'battleship',
            length: 4,
            spaces: [ 'A6', 'A7', 'A8', 'A9' ] 
        },
        { 
            name: 'cruiser', 
            length: 3, 
            spaces: [ 'A10', 'B10', 'C10' ] 
        },
        { 
            name: 'submarine', 
            length: 3, 
            spaces: [ 'D10', 'E10', 'F10' ] 
        },
        { 
            name: 'defender', 
            length: 2, 
            spaces: [ 'I10', 'H10' ] 
        } 
    ]

- Spaces needs to be an array with the length designated by the length key. Each item in the array must be an alphanumeric code with the first character being one of A-J, uppercase, the second/third must be a number 1-10.

`GET /api/games` Returns data on all active games for a user. It also expires any games that are more than 3 days old before returning the data. Returns data in the shape of:

    { 
        userId: 1,
        result:[ 
            { 
                id: 1,
                player1: 2,
                player2: 1,
                room_id: [UUID],
                next: null,
                turn: 'player1',
                game_status: 'active',
                player1_username: 'test-2',
                player2_username: 'test-1' 
            },
        ]
    }

- Result is an array full of objects representing each individual game a player is both in and is active. 

`GET /api/games/prev` Returns data on all inactive games for a user. These games are either completed, forfeited or expired. Returns data in the shape of: 

    { 
        userId: 1,
        playerUsername: 'example-username-1',
        result: [ 
            { 
                game_id: 4,
                player1_username: 'example-username-1',
                player2_username: 'example-username-2',
                game_status: 'complete',
                winner: 'player1' 
            },
        ]
    }

- Result is an array full of objects representing each individual game a player is both in and is inactive. 
- The game_status key is only ever one of the strings 'complete', 'forfeited' or 'expired'.

`GET /api/games/stats` Returns the stats for a logged in user. Returns data in the shape of:

    { 
        id: 1, 
        userid: 1, 
        wins: 2, 
        losses: 1, 
        username: 'example-username-1' 
    }

`GET /api/games/activegame/:gameId/:playerNum` Returns the game data for the specified gameId. Allows for the resuming of games. Returns data in the shape of: 

    { 
        player2_ships: false,
        player1_ships:[ 
            { name: 'aircraftCarrier', length: 5, spaces: [Array] },
            { name: 'battleship', length: 4, spaces: [Array] },
            { name: 'cruiser', length: 3, spaces: [Array] },
            { name: 'submarine', length: 3, spaces: [Array] },
            { name: 'defender', length: 2, spaces: [Array] } 
        ],
        player1_hits: [Array],
        player2_hits: [Array],
        player1_misses: [Array],
        player2_misses: [Array],
        currentUser: 'player1',
        id: 1,
        game_id: 1,
        turn: 'player1',
        winner: null,
        last_move: [Timestamp] 
    }

- The opponent's ships are not sent to the client at any point to prevent cheating. Instead a boolean is put in their place to indicate whether the opponent has set their ships yet. 

`PATCH /api/games/activegame/:gameId/:playerNum` Forfeits the game. This can only be done if the game is active. Whichever player forfeited recieves a loss and their opponent recieves a win. 

`GET /api/games/results/:gameId` Returns the results for a specified inactive game. Returns data in the shape of: 

    { 
        player1_hits: [Array],
        player1_misses: [Array],
        player2_hits: [Array],
        player2_misses: [Array],
        winner: 'player1'
    } 

___________
Errors


- `All Errors` will return with an error status and an object with an error key. The error message is written there. 

______

## Sockets

`Authorization` When connecting the socket must provide a valid bearer JSON web token to be authorized for socket useage. This socket connection can only be from the designated client's url. The Authorization header must be set in the extraHeaders for the client. 

Example client socket: 

    const socket = io(URL, {
        transportOptions: {
            polling: {
                extraHeaders: {
                    'Authorization': `Bearer ${some-jwt-here}`
                }
            }
        }
    });

___

`On 'join_room'` Socket must provide either the string 'random' or a valid UUID pertaining to the room_id of a game they are a part of. If 'random' is provided they are connected to a game in the queue if there is one otherwise they are connected to a random room and put in the queue. 

This allows for the pairing of client's in a match. One cannot play against themselves nor have more than 10 active games. 

If 'random' is provided the socket will be notified through the `joined` channel. Otherwise if a room id is provided the socket will be notified through the `reconnected` channel. 

___

`On 'fire'` Socket must provide an object with keys for the target they wish to fire upon, the game id and room id. 

This will check to see if the socket's shot was a hit/miss and notify both users through the `response` channel. If the socket's shot wins the game both users will be notified through the `win` channel. 

Example object from client: 

    {
        target: 'A1',
        gameId: 1,
        roomId: [UUID]
    }

___

`On 'ships_ready'` Used to notify the other user that your ships have been set. The other user in the room will be notified through the `opponent_ready` channel. 
___

`On 'send-message'` Used to send a message to the other user in the room. This allows for chat functionality. Socket must provide an object with keys for the room and message they wish to send. 

The other user will be notified through the `chat-message` channel.

Example object from client: 

    {
        room: [UUID],
        message: 'This is the message.'
    }

___

## Summary

This server allows for the connection of two users to facilitate playing the classic boardgame Battleship. 

They are matched against other Users through the use of websockets and express routing. The gameplay is actively updated so players can play in real time. 

The data is stored in a PostgreSQL database which allows users to player passively agaisnt their opponent as well. 

## Technology Used

Javascript, PostgreSQL, Express, Mocha, Chai, Socket .io

## Programmers

- Aedan Warfield - Project Manager - https://www.linkedin.com/in/aedanwarfield/
- Shannon Lichtenwalter - Product Owner - https://www.linkedin.com/in/shannon-lichtenwalter/
- Sean Cooper - Quality Assurance - https://www.linkedin.com/in/sean-cooper-20799a185/
- Heesu Kang - CSS Lead - https://www.linkedin.com/in/heesu-kang/