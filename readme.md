For the purposes of this specification, the complete API for accessing scores, user data and other information from a remote eAmusement network will be referred to as "the data portability API" or just "the API." The version of this spec is currently "v1". When specifying attribute names or values, the case-sensitive, valid values will be presented either in "quotes" or in ``monospace``.

# Wire Protocol

The API should be implemented as an HTTP service, utilizing only GET requests as it is read-only. Due to buggy client libraries in some languages, such as cURL, the API can also respond to POST requests in an identical fashion to GET requests. Any request parameters will be passed via the request body as a JSON object. The mime type of every request should be "application/json". Requests that have no request parameters should send an empty JSON object ({}). All requests should be authenticated against an API token previously generated by the server, placed in the "Authorization" http header with the type "Token". API tokens can be any arbitrary ASCII string (without spaces or control characters) but should be at least 1 character in length and at most 64 characters in length. All response data will be passed via the response body as a JSON object. The mime type of every response should be "application/json". The content encoding for all requests and responses should be "utf-8". On error, the mime type and content type should still be set and a JSON object with an "error" attribute describing the error should be returned. The API should use HTTPS to protect the authorization token and request/response data. HTTP features such as eTags and compression are optional, and servers should not require clients to support them, or vice versa. It is recommended to leave handling for such features up to the webserver.

An example request and response looks as such:

**Request**

    GET / HTTP/1.1
    Authorization: Token deadbeef
    Content-Type: application/json; charset=utf-8

    {}

**Response**

    HTTP/1.1 200 OK
    Content-Type: application/json; charset=utf-8

    {"versions":["v1"],"name":"Test Server"}

An example error request and response looks as such:

**Request**

    GET /invalid/uri HTTP/1.1
    Authorization: Token deadbeef
    Content-Type: application/json; charset=utf-8

    {}

**Response**

    HTTP/1.1 405 Method Not Allowed
    Content-Type: application/json; charset=utf-8

    {"error": "The URI /invalid/uri was not recognized!"}

Valid status codes are as follows (essentially identical to their meaning in traditional http):

* 200 - Request okay, response follows. Return this when a valid lookup has occurred and response data should be considered valid, even in the case of no data available for the request.
* 401 - Unauthorized token. Return this when the token is not provided, invalid or has been revoked.
* 404 - Game/version combination unsupported. Return this when the requesting URI is valid but the server does not support this game/version, or when a valid object type is requested that isn’t supported by this server.
* 405 - URI not allowed. Return this when an invalid URI or invalid method is requested.
* 500 - Uncaught server error. Return this on unexpected server-side problems.
* 501 - Unimplemented. Return this when a client requests a version of the API that the server does not support.

## URI Structure

The URI structure of the API will follow this format:

    /<protocol version>/<game series>/<game version>

* Protocol Version - The version of this API spec the server conforms to. Currently this should be hardcoded to "v1". The purpose of this is to allow for non-backwards-compatible changes to the API in the future while not breaking existing clients.
* Game Series - The series this request is for. This works in tandem with the game version below to specify a distinct set of objects to look up (such as valid songs).
* Game Version - The version of the game this request is for. This depends on the game series being requested and is implied to be the final (latest) version of data for that particular game/version combo. Note that this is not an integer, even though in most cases it can be cast to one!

Note that some servers support games that have had songs added back in via hacks, such as IIDX Omnimix. To request data from an omni version of a game, prepend the letter "o" to the version. Servers may treat this how they wish (they may wish to strip the leading o and return only info for the original game or return 404 in the instance they don’t support this version).

## Request/Response Format

Each request should have a JSON object that specifies the "ids", "type" and "objects" parameters, as attributes. The "ids" attribute should be a list of zero or more string IDs, as interpreted by the "type" attribute. Note that the ordering for this list is important, objects with more than one part to their ID (such as the 'song' type) will have different meaning for each part of the ID. The "type" attribute should be a string, with valid values documented below. The "objects" attribute should be a list of zero or more string objects being fetched, as documented in the Supported Objects section below. Note that there is only one "type" for all IDs requested, regardless of how many objects are being fetched. This is intentional as the request is constrained to looking up information based on one particular index to keep the response format from getting complicated. If the wrong number of list entries for an ID is requested, the server should return a 405 error code. If an object is requested with an ID that makes no sense (such as requesting a profile based on song ID), a 405 error code should be returned. If an unrecognized object is requested, the server should return a 404 error code. If an unrecognized type is requested, the server should return a 404 error code. If a request is made with zero objects, a 405 error code should be returned. No attempt to return partial data should be made when part of the request makes sense but part doesn't, or when part of the request causes an exception.

The following ID types are supported for retrieving various objects from a remote server.

* ``card`` - 16 character hex string as found when using a slotted or wavepass reader to read an eAmusement card. A client can request a list of one or more card IDs, and the response should include all information for that object type for each card IDs. In most cases, an empty response for card IDs that do not exist on the server being queried is sufficient.
* ``song`` - A game identifier for a particular song (usually an integer when talking to a game). In many cases this only makes sense in the context of a game/version so be sure to take all three into account when looking up objects by song. Optionally, a second value in the list can be provided specifying the chart for a song. Note that while these are normally integers, they should be passed to the server as strings. Unlike ‘card’ lookups, this only allows looking up by one song or song/chart object at a time.
* ``server`` - No ID necessary, fetches all instances of a particular object on the server for a given game/version. An empty list should be given to the "ids" parameter.

On successful lookup, the response JSON object will have an attribute for each requested object type, named after that object request. The value of each attribute will depend on what's being looked up but could be a list (in the case of looking up records, for example) or a JSON object itself. Valid responses for each object type will be documented below in the Supported Objects section.

An example request for profile and records based on a card is as follows:

**Request**

    GET /v1/iidx/24 HTTP/1.1
    Authorization: Token deadbeef
    Content-Type: application/json; charset=utf-8

    {"ids":["E0040000DEADBEEF"],"type":"card","objects":["profile","records"]}

**Response**

    HTTP/1.1 200 OK
    Content-Type: application/json; charset=utf-8

    {"profile":[],"records":[]}

An example request for records based on a particular song/chart is as follows:

**Request**

    GET /v1/iidx/24 HTTP/1.1
    Authorization: Token deadbeef
    Content-Type: application/json; charset=utf-8

    {"ids":["1001", "1"],"type":"song","objects":["records"]}

**Response**

    HTTP/1.1 200 OK
    Content-Type: application/json; charset=utf-8

    {"records":[]}

An example request for all records on the server for a game/series is as follows:

**Request**

    GET /v1/iidx/24 HTTP/1.1
    Authorization: Token deadbeef
    Content-Type: application/json; charset=utf-8

    {"ids":[],"type":"server","objects":["records"]}

**Response**

    HTTP/1.1 200 OK
    Content-Type: application/json; charset=utf-8

    {"records":[]}

## Supported Games

Valid game series and their versions are as follows. Clients and servers should use the following game/version combinations to identify the objects being requested.

* ``beatstream``
    * ``1`` - BeatStream
    * ``2`` - BeatStream アニムトライヴ
* ``danceevolution``
    * ``1`` - DanceEvolution
* ``ddr``
    * ``9`` - DanceDanceRevolution SuperNova
    * ``10`` - DanceDanceRevolution SuperNova 2
    * ``11`` - DanceDanceRevolution X
    * ``12`` - DanceDanceRevolution X2
    * ``13`` - DanceDanceRevolution X3 VS 2ndMIX
    * ``14`` - DanceDanceRevolution 2013
    * ``15`` - DanceDanceRevolution 2014
    * ``16`` - DanceDanceRevolution Ace
* ``drummania``
    * TBD, please fill in logical mapping!
* ``futuretomtom``
    * ``1`` - ミライダガッキ FutureTomTom
* ``guitarfreaks``
    * TBD, please fill in logical mapping!
* ``iidx``
    * ``18`` - IIDX 18 Resort Anthem
    * ``19`` - IIDX 19 Lincle
    * ``20`` - IIDX 20 Tricoro
    * ``21`` - IIDX 21 SPADA
    * ``22`` - IIDX 22 PENDUAL
    * ``23`` - IIDX 23 copula
    * ``24`` - IIDX 24 SINOBUZ
    * ``25`` - IIDX 25 CANNON BALLERS
* ``jubeat``
    * ``1`` - Jubeat
    * ``2`` - Jubeat Ripples
    * ``2a`` - Jubeat Ripples Append
    * ``3`` - Jubeat Knit
    * ``3a`` - Jubeat Knit Append
    * ``4`` - Jubeat Copious
    * ``4a`` - Jubeat Copious
    * ``5`` - Jubeat Saucer
    * ``5a`` - Jubeat Saucer Fulfill
    * ``6`` - Jubeat Prop
    * ``7`` - Jubeat Qubell
    * ``8`` - Jubeat Clan
* ``museca``
    * ``1`` - MUSECA
    * ``1p`` - MUSECA 1+1/2
* ``nostalgia``
    * ``1`` - ノスタルジア
    * ``2`` - ノスタルジア FORTE
* ``popnmusic``
    * ``19`` - Pop’n Music Tune Street
    * ``20`` - Pop’n Music Fantasia
    * ``21`` - Pop’n Music Sunny Park
    * ``22`` - Pop’n Music Lapistoria
    * ``23`` - Pop’n Music Eclale
    * ``24`` - Pop’n Music うさぎと猫と少年の夢
* ``reflecbeat``
    * ``1`` - REFLEC BEAT
    * ``2`` - REFLEC BEAT limelight
    * ``3w`` - REFLEC BEAT colette -Winter-
    * ``3sp`` - REFLEC BEAT colette -Spring-
    * ``3su`` - REFLEC BEAT colette -Summer-
    * ``3a`` - REFLEC BEAT colette -Autumn-
    * ``3as`` - REFLEC BEAT colette -All Seasons-
    * ``4`` - REFLEC BEAT groovin’!!
    * ``4u`` - REFLEC BEAT groovin’!! Upper
    * ``5`` - REFLEC BEAT VOLZZA
    * ``6`` - REFLEC BEAT VOLZZA 2
    * ``7`` - REFLEC BEAT 悠久のリフレシア
* ``soundvoltex``
    * ``1`` - Sound Voltex BOOTH
    * ``2`` - Sound Voltex II Infinite Infection
    * ``3`` - Sound Voltex III Gravity Wars
    * ``4`` - Sound Voltex IV Heavenly Haven

## Special URIs

While normal requests will follow the exact pattern above, special URIs documented below should also be supported.

### API Information URI

    /

The API Information URI should return information about the server itself. The request JSON should be an empty object. The returned JSON object should contain the following attributes:

* ``versions`` - A list of strings of supported protocol versions that this server supports. Right now, this should be a list of one string, "v1".
* ``name`` - The name of the network that we are talking to. This can be displayed on the admin panel of a client or used in a basic query to verify functionality.
* ``email`` – An administrative email used to contact the network if the need arises.

# Supported Objects

The following objects are supported for fetching from the API. As documented above, each request object should be returned as an attribute named after itself in the response JSON, containing either a list of that object type, or a JSON object itself representing the response.

## Records

Identified by the "records" object type, this pulls up high scores based on the object type. For "card", it pulls up all high scores for the game/version associated with the user that owns the card. When multiple card IDs are presented, it pulls up all high scores for each user associated with each card. In the case that the user does not exist, an empty list should be returned to represent that the user does not have any scores on this network. It is expected that there will be at most one high score per song/chart/user, since a single user can only have a single high score on each song/chart. For "song", it pulls up all high scores for a game/version/song. Optionally, if the chart is provided, constrains the search to only that chart instead of returning records for all charts. For "server", it pulls up the record for each song/chart that the server recognizes for that game/version.

Aside from basics such as the points achieved, each game will have different attributes required, as documented in game-specific sections below. The response to the records request should be a JSON object with a "records" attribute. The "records" attribute should be a list of zero or more record objects that fit the criteria of the object type. Each record object should have the following attributes at minimum:

* ``card`` – The 16 character card ID identifying this user as a string. In the case that a user has multiple cards associated with an account, the card ID used to request records from the user should be returned. For 'song' and 'server' ID lookups, this should return the first card registered to the account.
* ``song`` – The game identifier for the song this record belongs to as a string. May be version specific, so parsing it is up to the client based on the requested game/version.
* ``chart`` – The game identifier for the chart this record belongs to as a string. May be version specific as above.
* ``points`` – The number of points earned for this record as an integer. 
* ``timestamp`` – The integer UTC unix timestamp when the record was earned.
* ``updated`` – The integer UTC unix timestamp when the record was last updated. This includes incrementing the play count, even if everything else stays the same.
* ``plays`` – The integer number of plays logged for this song/chart for the given ID.

Given that a high score, once set, is immutable, records are easily cached on the client. For that reason, it is important that a new record earned on a server change the timestamp to the time that new record was earned. In that way, clients can request only records earned since a particular timestamp in order to download incremental changes since last fetch given a particular ID.

### Optional Request Parameters

* ``since`` – A UTC unix timestamp to constrain the range of records looked up and returned. If provided, records with an update time greater than or equal to this time should be returned, and records less than this time should be excluded.
* ``until`` – A UTC unix timestamp to constrain the range of records looked up and returned. If provided, records with an update time less than this time should be returned, and records greater than or equal to this time should be excluded.

### DDR Additional Attributes and Documentation

Valid charts for DDR according to the game are as follows:

* ``0`` – Single beginner
* ``1`` – Single basic
* ``2`` – Single difficult
* ``3`` – Single expert
* ``4`` – Single challenge
* ``5`` – Double basic
* ``6`` – Double difficult
* ``7`` – Double expert
* ``8`` – Double challenge

The following attribute should be returned (if available) for records belonging to the DDR series.

* ``rank`` – The letter grade ranking that the user has earned, as a string enum. Should be one of the following values: "AAA", "AA+", "AA", "AA-", "A+", "A", "A-", "B+", "B", "B-", "C+", "C", "C-", "D+", "D", "E".
* ``halo`` – The combo halo that the user has earned, as a string enum. Should be one of the following values:
    * ``none`` – The user did not earn a halo (no combo).
    * ``gfc`` – Good full combo.
    * ``fc`` – Great full combo.
    * ``pfc`` – Perfect full combo.
    * ``mfc`` – Marvelous full combo.
* ``combo`` – An integer specifying the maximum combo earned.
* ``ghost`` - List of integers representing the integers that the game sends for ghost steps for a song. Note that DDR A changes this format to be a string of single digit integers, so a transformation will need to be done to fulfill this. Note that ghost steps should not be considered compatible across versions of DDR for this reason.

### IIDX Additional Attributes and Documentation

Valid charts for IIDX according to the game are as follows:

* ``0`` – Normal 7key
* ``1`` – Hyper 7key
* ``2`` – Another 7key
* ``3`` – Normal 14key
* ``4`` – Hyper 14key
* ``5`` – Another 14key
* ``6`` – Beginner 7key

Note that in the case of IIDX, "points" above refers to the EX score earned on the song. This is calculated as 2 ✕ pgreats + greats. The following attributes should be returned (if available) for records belonging to the IIDX series.

* ``status`` – String enum representing the clear lamp earned on this particular record. Valid values are as follows:
    * ``np`` – no play/no clear (such as when earning a record during DAN courses).
    * ``failed`` – Failed song.
    * ``ac`` – Assist clear.
    * ``ec`` – Easy clear.
    * ``nc`` – Normal clear.
    * ``hc`` – Hard clear.
    * ``exhc`` – EX hard clear.
    * ``fc`` – Full combo.
* ``ghost`` – List of integers in the range of 0-255, representing the bytes that the game sends for ghost steps for a song.
* ``miss`` – The miss count, as an integer number of misses recorded for this record. If the miss count is not available for this song (such as the user failed out early), this should be -1. Otherwise, the valid range goes from 0 to the number of notes in the chart.
* ``pgreat`` - The count of pgreats that the user earned for this particular record. If not available, this should be -1.
* ``great`` - The count of greats that the user earned for this particular record. If not available, this should be -1.

### Jubeat Additional Attributes and Documentation

Valid charts for Jubeat according to the game are as follows:

* ``0`` – Basic
* ``1`` – Advanced
* ``2`` – Extreme

The following attributes should be returned (if available) for records belonging to the Jubeat series.

* ``status`` – The clear medal earned by the user for this record, as a string enum. Valid values are as follows:
    * ``failed`` – The user failed the song.
    * ``cleared`` – The user cleared the song with no combo.
    * ``nfc`` – Nearly full combo. The user got almost a full combo.
    * ``fc`` – Full combo.
    * ``nec`` – Nearly excellent combo. The user got close to perfect on each note.
    * ``ec`` – Excellent combo. The user earned an excellent on every note.
* ``combo`` – Integer specifying the maximum combo earned when the user earned the record.
* ``ghost`` – List of integers representing the ghost data sent by the game for this record.

### Museca Additional Attribute and Documentation

The following chart types are recognized by Museca according to the game:

* ``0`` – Green chart.
* ``1`` – Orange chart.
* ``2`` – Red chart.

The following attributes should be returned (if available) for all records belonging to the Museca series:

* ``status`` – String enum representing the clear status of this record. Valid values are as follows:
    * ``failed`` – The user failed the song for this record.
    * ``cleared`` – The user cleared the song for this record.
    * ``fc`` – The user earned a full combo for this record.
* ``rank`` – String enum representing the ranking earned during this record. Valid values are as follows. Note that these correspond one-to-one with the kanji rank displayed in the UI, so they are reproduced below to avoid translation errors:
    * ``death`` - 没
    * ``poor`` - 拙
    * ``mediocre`` - 凡
    * ``good`` - 佳
    * ``great`` - 良
    * ``excellent`` - 優
    * ``superb`` - 秀
    * ``masterpiece`` - 傑
    * ``perfect`` - 傑 (Identical to masterpiece kanji but shows up gold in-game).
* ``combo`` - The maximum combo earned when earning this record as an integer.
* ``buttonrate`` – The integer button rate according to the game when this score was recorded.
* ``longrate`` – The integer hold rate according to the game when this score was recorded.
* ``volrate`` – The integer spin rate according to the game when this score was recorded.

### Pop’n Music Additional Attributes and Documentation

The following chart types are recognized by Pop’n Music Fantasia and above:

* ``0`` – Easy chart
* ``1`` – Normal chart
* ``2`` – Hyper chart
* ``3`` – EX chart

Pop’n Music Tune Street only had ‘cool’ timing for a particular mode (Cho Challenge mode), and thus only those scores are compatible going forward. The charts that were available in this mode are the "normal", "hyper" and "EX" chart types. Easy charts were not available, so scores should not be loaded for easy charts for Tune Street.

The following attributes should be returned (if available) for records belonging to the Pop’n Music series:

* ``status`` – The clear status when this record was earned, as a string enum. Valid values are as follows:
    * ``cf`` – Failed, circle marker.
    * ``df`` – Failed, diamond marker.
    * ``sf`` – Failed, star marker.
    * ``ec`` – Easy cleared.
    * ``cc`` – Cleared, circle marker.
    * ``dc`` – Cleared, diamond marker.
    * ``sc`` – Cleared, star marker.
    * ``cfc`` – Full combo cleared, circle marker.
    * ``dfc`` – Full combo cleared, diamond marker.
    * ``sfc`` – Full combo cleared, star marker.
    * ``p`` – Perfect full combo cleared.
* ``combo`` – Integer specifying the maximum combo earned when this record was stored.

### Reflec Beat Additional Attributes and Documentation

The following charts are recognized by Reflec Beat series. Note that special/white hard charts do not appear at all in some older series, so they shouldn’t be returned when requesting records for older games.

* ``0`` – Basic
* ``1`` – Medium
* ``2`` – Hard
* ``3`` – Special/White Hard

Note that the game went through a complete redesign between Volzza 2 and Reflesia, and thus scores are not compatible across this boundary in either direction.

The following attributes should be returned (if available) for any record belonging to the Reflec Beat series:

* ``rate`` – Achievement rate of this record. This should be an integer value where 0 is 0% and 10000 is 100% achievement rate. Essentially, it is the achievement rate percentage multiplied by 100.
* ``status`` – Clear status according to the game as a string enum. Note that some versions conflate clear status and combo type, so some separating and combining might need to be done depending on how a network decides to store attributes. Valid values are as follows:
    * ``np`` – No play. Similar to IIDX, where the record was earned in a mode that does not credit the user for playing the song.
    * ``failed`` – The user failed the song for this record.
    * ``cleared`` – The user cleared this song for this record.
    * ``hc`` – The user hard cleared this song.
    * ``shc`` – The user S hard cleared the song.
* ``halo`` – The combo status that was achieved when earning this record, as a string enum. Valid values are as follows:
    * ``none`` – The user did not earn any combo marker during this record.
    * ``ac`` – Almost combo. The user was close to a full combo but missed by up to two.
    * ``fc`` – Full combo.
    * ``fcaj`` – Full combo all just reflec. The user was able to full combo the song, and on top of that execute the maximum number of just reflecs available in the song.
* ``miss`` – The miss count earned during this song, as an integer. If this is not available (the user earned this song in a mode that doesn’t count misses, for instance), this should be set to -1.
* ``combo`` – The combo earned during this song as an integer.

### SDVX Additional Attributes and Documentation

The following chart types are recognized by SDVX according to the game:

* ``0`` – Novice
* ``1`` – Advanced
* ``2`` – Exhaust
* ``3`` – Infinite
* ``4`` – Maximum

The following attributes should be returned (if available) by all records belonging to the SDVX series.

* ``status`` – The clear status of this record, as a string enum. Valid values are as follows:
    * ``np`` – No play/no clear, such as when playing during a mode that does not give you credit for clears but still saves scores (such as skill analyzer).
    * ``cleared`` – User cleared the chart for this record.
    * ``hc`` - User hard cleared the chart for this record.
    * ``uc`` – User full combo’d the chart (ultimate chain) for this record.
    * ``puc`` – User perfected the chart (perfect ultimate chain) for this record.
* ``rank`` – The clear rank that was earned by the user for this record, as a string enum. Valid values are as follows: "E" (failed, no play), "D", "C", "B", "A", "A+", "AA", "AA+", "AAA", "AAA+" and "S".
* ``combo`` – The maximum combo earned by the user for this record as an integer.
* ``buttonrate`` – The integer button rate according to the game when this score was recorded.
* ``longrate`` – The integer hold rate according to the game when this score was recorded.
* ``volrate`` – The integer knob volume rate according to the game when this score was recorded.

## Profile

Identified by the "profile" object type, this pulls up basic profile information for users. For "card", it pulls up the profile for the game associated with the user that owns the card. The version requested should be preferred, but if no profile exists for that exact game/version a compatible profile for another version of the game should be returned. When multiple card IDs are presented, it pulls up the profile for the user associated with each card. In the case that the user does not exist, an empty list should be returned to represent that the user does not have any profile on this network for this game. If a client is looking up multiple profiles and some are present and others are not, the returned list of profiles should only include present profiles. The client can determine that the profile for a user doesn’t exist by the absence of a profile object for that card in the return. For "song", a 405 error code should be returned to specify that this is a nonsense request. For "server", it pulls up the profile for each user that the server recognizes for that game/version. Exact matches should be returned only.

The profile request isn’t currently meant to allow instantiation of full game profiles across different networks. Instead, it is provided as a way to look up in-game name and statistic information based on card ID so networks can present global scores or allow server-server rivals if desired. Given the amount of information in the profile return for each user, a server may also wish to present to the user a profile migration screen as if the user was coming from an older version of the game. In this way, the name on the profile can be pre-filled from another network as a convenience. The response to the profile request should be a JSON object with a "profile" attribute. The "profile" attribute should be a list of zero or more profile objects that fit the criteria of the object type. Each profile object should have the following attributes at minimum:

* ``card`` – The 16 character card ID identifying this user as a string. In the case that a user has multiple cards associated with an account, the card ID used to request this user’s profile should be returned.
* ``name`` – The name associated with the profile, as a string.
* ``registered`` – The integer UTC unix timestamp of profile registration. If unavailable, this should be set to -1.
* ``updated`` – The integer UTC unix timestamp of the last profile update. If unavailable, this should be set to -1.
* ``plays`` – The integer total number of plays this user has logged using this profile. If unavailable, this should be set to -1.
* ``match`` - A string enum representing whether this profile was an exact match or a partial match. Valid values are as follows:
   * ``exact`` - This profile is for the requested game/version. Additional attributes specified below are for this game/version.
   * ``partial`` - This profile is for the requested game, but a different version. If the server is capable of doing so, additional attributes should be converted for corret consumption for the game/version requested by the client. If it is not possible, they should be set to -1 to indicate they are not available for the requested game/version.

### DDR Additional Attributes and Documentation

The following attributes should be returned (if available) by all profiles belonging to the DDR series.

* ``area`` - The integer area code as set when creating a profile on DDR. If unavailable, this should be set to -1.

### IIDX Additional Attributes and Documentation

The following attributes should be returned (if available) by all profiles belonging to the IIDX series.

* ``area`` - The integer prefecture code as set when creating a profile on IIDX. If unavailable, this should be set to -1.
* ``qpro`` - An object representing the player's qpro, with the following attributes:
  * ``head`` - The integer representation of the game's current setting for qpro head. If unavailable, this should be set to -1.
  * ``hair`` - The integer representation of the game's current setting for qpro hair. If unavailable, this should be set to -1.
  * ``face`` - The integer representation of the game's current setting for qpro face. If unavailable, this should be set to -1.
  * ``body`` - The integer representation of the game's current setting for qpro body. If unavailable, this should be set to -1.
  * ``hand`` - The integer representation of the game's current setting for qpro hand. If unavailable, this should be set to -1.

### Jubeat Additional Attributes and Documentation

Currently Jubeat has no additional attributes that are recognized by the API.

### Museca Additional Attribute and Documentation

Currently Museca has no additional attributes that are recognized by the API.

### Pop’n Music Additional Attributes and Documentation

The following attributes should be returned (if available) by all profiles belonging to the Pop'n Music series.

* ``character`` - The integer character code as set when creating a profile on Pop'n Music. If unavailable, this should be set to -1.

### Reflec Beat Additional Attributes and Documentation

Currently Reflec Beat has no additional attributes that are recognized by the API.

### SDVX Additional Attributes and Documentation

Currently SDVX has no additional attributes that are recognized by the API.
