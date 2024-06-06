# Rockset Recipe: Quickstart Guide

This is a step-by-step guide on how to execute the [Rockset Quickstart](https://docs.rockset.com/documentation/docs/quickstart) using the Python client. This Quickstart involves creating Film Releases and Film Ratings collections from a public S3 bucket and then executing queries to get recommended movies based on a specified genre and rating value.

## Requirements
- Python >= 3.6
- `pip install rockset` or `pip3 install rockset`

*_The only things you have to change to successfully run this script are **rocksetApiKey** and **apiServerHost**_

## Step 1: Initialize the Client

Before we can start, we'll need to initialize the Rockset client. Create an API key in the [API Keys tab of the Rockset Console](https://console.rockset.com/apikeys). The region can be found in the dropdown menu at the top of the page. If you don't already have a Rockset account, click [here](https://rockset.com/create/) to create a FREE account with $300 credits.

```
# Set the API Key as an environmental variable in the shell via:
# export ROCKET_API_KEY='<insert key here>'
rocksetApiKey = os.getenv("ROCKSET_API_KEY")
apiServerHost = Regions.YOUR_REGION_HERE #ex: Regions.usw2a1

rs = RocksetClient(host=apiServerHost, api_key=rocksetApiKey)
```

## Step 2: Define Parameters

First we define parameters for the [Workspace](https://docs.rockset.com/documentation/docs/workspaces) the collections will exist in, the collection names, and prefixes required to get collection data from public S3 buckets.

```
# Define Workspace name
workspaceName = 'movies'

# Define Collection names
movieReleasesCollectionName = 'film_releases'
movieRatingsCollectionName = 'film_ratings'

# Define prefixes for public S3 bucksets when creating Collections
movieReleasesPrefix = 'movies'
movieRatingsPrefix = 'movie-ratings'
```

## Step 3: Define Ingest Transformations

We will next define [Ingest Transformations](https://docs.rockset.com/documentation/docs/ingest-transformation) for our collections. Ingest Transformations are SQL queries applied to data _before_ it is stored in Rockset.

```
# Ingest transformation for `film_releases` collection
filmReleasesIngestTransformation = """
    SELECT
        _input._meta,
        NULLIF(_input.belongs_to_collection, '') AS belongs_to_collection,
        _input.genres,
        NULLIF(_input.homepage, '') AS homepage,
        TRY_CAST(_input.id AS int) AS id,
        _input.original_language,
        _input.overview,
        _input.popularity,
        _input.production_companies,
        CAST(_input.release_date AS date) AS release_date,
        _input.revenue,
        _input.runtime,
        _input.title,
        _input.vote_average,
        _input.vote_count
    FROM
        _input;
    """
```

```
# Ingest transformation for `film_ratings` collection
filmRatingsIngestTransformation = """
    SELECT
        _meta,
        TRY_CAST(_input.movieId AS int) AS movie_id,
        _input.rating,
        TRY_CAST(userId AS int) AS user_id
    FROM
        _input;
    """
```

## Step 5: Create Workspace

First, we'll create a `movie_data` [Workspace](https://docs.rockset.com/documentation/docs/workspaces). It is good practice to separate different use cases into different workspaces.

```
def create_workspace(rs, workspaceName):
    try:
        print(f"Creating workspace `{workspaceName}`...")
        api_response = rs.Workspaces.create(
            name=workspaceName,
        )
    except ApiException as e:
        print("Exception when creating workspace: %s\n" % json.loads(e.body))
```

```
create_workspace(rs, workspaceName)
```

## Step 6: Create Collections

Next, we'll create a `Film Releases` [Collection](https://docs.rockset.com/documentation/docs/collections) and a `Film Ratings` Collection from a public S3 bucket. We'll then pass our ingest transformations from the previous steps into the field_mapping_query.

```
def create_collection(rs, workspaceName, collectionName, sourcePrefix, ingest_transformation_query):
    try:
        print(f"Creating collection `{collectionName}`...")
        api_response = rs.Collections.create_s3_collection(
            field_mapping_query=FieldMappingQuery(
                sql=ingest_transformation_query,
            ),
            name=collectionName,
            workspace=workspaceName,
            sources=[
                S3SourceWrapper(
                    bucket="s3://rockset-public-datasets",
                    prefix=sourcePrefix,
                    region="us-west-2",
                ),
            ]
        )
    except ApiException as e:
        print("Exception when creating collection: %s\n" % json.loads(e.body))
```

```
# Create Film Releases Collection
create_collection(rs, workspaceName, movieReleasesCollectionName, movieReleasesPrefix, filmReleasesIngestTransformation)

# Create Film Ratings Collection
create_collection(rs, workspaceName, movieRatingsCollectionName, movieRatingsPrefix, filmRatingsIngestTransformation)
```

### Wait for collections to be ready

Before we can execute queries, we will need to wait for the collections to be in the `READY` state. It will only take ~1 minute to ingest all the movie data from the public S3 buckets.

```
def wait_for_collection_ready(rs, workspaceName, collectionName, max_attempts=30):
    print(f"Waiting for the `{collectionName}` collection to be `Ready` (~1 minute)...")
    for attempt in range(max_attempts):
        api_response = rs.Collections.get(collection=collectionName, workspace=workspaceName)

        if api_response.data.status == 'READY':
            print(f"Collection `{collectionName}` ready!")
            break
        else:
            time.sleep(60)
```

```
wait_for_collection_ready(rs, workspaceName, movieReleasesCollectionName)
wait_for_collection_ready(rs, workspaceName, movieRatingsCollectionName)
```

## Step 7: Movie Suggestions Query

### Define Query

We will now define a query that suggests movies to a user based on their genre preference and the movie's rating. Since `genre` is an array field (as a single movie may fit multiple genres), we use [`UNNEST`](https://docs.rockset.com/documentation/reference/select#unnest) to expand this array and create a record for each `(genre, movie)` pair. We'll also choose to exclude movies rated by a specified user.

```
movie_suggestions_query = f"""
SELECT
    m.id,
    m.title
FROM
    {workspaceName}.{movieReleasesCollectionName} m,
    UNNEST(m.genres) as genres
WHERE
    genres.name = 'Action'
    AND m.id NOT IN (
        SELECT
            r.movie_id
        FROM
            {workspaceName}.{movieRatingsCollectionName} r
        WHERE
            r.user_id = 100
    )
ORDER BY
    m.popularity DESC;
"""
```

### Execute Query

With our environment all setup, executing this query is just a simple API call!

```
execute_query(rs, movie_suggestions_query)
```

## Step 8: Movie Suggestions Query with Parameters

### Define Query

In the query from the previous step, we used the `Action` genre and user_id `100`. Now, let's make these values parameters that can be specified at runtime.

First we'll need to define an array of parameters. We'll define one with name `genre` of type `string` with value `Action` and one with name `user_id` of type `int` with value `100`. 
```
parameters=[
    QueryParameter(
                name="genre",
                type="string",
                value="Action",
            ),
    QueryParameter(
                name="user_id",
                type="int",
                value="100",
            )
]
```

Next we'll modify the SQL statement from the previous step to incorporate these parameters:
- Replace `genres.name = 'Action'` with `genres.name = :genre`.
- Replace `r.user_id = 100` with `r.user_id = :user_id`.

```
movie_suggestions_parameters_query = f"""
SELECT
    m.id,
    m.title
FROM
    {workspaceName}.{movieReleasesCollectionName} m,
    UNNEST(m.genres) as genres
WHERE
    genres.name = :genre
    AND m.id NOT IN (
        SELECT
            r.movie_id
        FROM
            {workspaceName}.{movieRatingsCollectionName} r
        WHERE
            r.user_id = :user_id
    )
ORDER BY
    m.popularity DESC;
"""
```

### Execute Query

Executing this query is just another simple API call!

```
execute_query(rs, movie_suggestions_parameters_query, parameters)
```

## Step 9: Clean Up

That completes the Rockset Quickstart Guide! As a courtesy to you, I've included a function to delete the workspace and collections that were created at your leisure.

```
def clean_up_demo(rs, workspaceName, collectionNames, max_attempts=10):

    # Deleting Collections
    for collectionName in collectionNames:
        try:
            print(f"Deleting collection `{collectionName}`...")
            api_response = rs.Collections.delete(
                collection=collectionName, 
                workspace=workspaceName
            )
        except ApiException as e:
            print(f"Exception when deleting collection: %s\n" % json.loads(e.body))
    # Checking if Collection is deleted
    for collectionName in collectionNames:
        for attempt in range(max_attempts):
            try:
                api_response = rs.Collections.get(
                    collection=collectionName, 
                    workspace=workspaceName
                )
            except ApiException as e:
                if json.loads(e.body).get("message_key") == "COLLECTION_DOES_NOT_EXIST": 
                    print(f"Collection `{collectionName}` deleted.")
                    break
            time.sleep(30)

    # Deleting Workspace
    try:
        print(f"Deleting workspace `{workspaceName}`...")
        api_response = rs.Workspaces.delete(
            workspace=workspaceName
        )
        # Checking if Workspace is deleted
        for attempt in range(max_attempts):
            try:
                api_response = rs.Workspaces.get(
                    workspace=workspaceName
                )
            except ApiException as e:
                if json.loads(e.body).get("message_key") == "WORKSPACE_NOT_FOUND": 
                    print(f"Workspaces `{workspaceName}` deleted.")
                    break
            time.sleep(30)
    except ApiException as e:
        print(f"Exception when deleting workspace: %s\n" % json.loads(e.body))
```

```
clean_up_demo(rs, workspaceName, [movieReleasesCollectionName, movieRatingsCollectionName])
```

## What's Next?

This completes the Quickstart tutorial! Here are some suggestions for next steps:

1. Keep building by checking out some of the pages below to continue exploring Rockset:
- [Connect your external data source](https://docs.rockset.com/documentation/docs/data-sources)
- [Explore the SQL Reference](https://docs.rockset.com/documentation/reference/all-commands)
- [Explore the REST API Reference](https://docs.rockset.com/documentation/reference/rest-api)
- [Check out our Developer Tools](https://docs.rockset.com/documentation/docs/libraries-tools)

2. Invite other members of your team to your organization in the [Users tab of the Rockset Console](https://console.rockset.com/users). You can determine if each new user should have Administrator, Member, or Read-Only access to Rockset. Check out our docs on [User Management](https://docs.rockset.com/documentation/docs/identity-access-management).

3. Check out the [Rockset community](https://community.rockset.com/) and share what you are looking to build with Rockset. We’re hanging out and ready to answer your questions!
