### What this script does
This script takes data from a Google spreadsheet, converts it to a JSON array
where each object has keys corresponding to the spreadsheet column names, then
then stores that JSON locally and/or automatically commits it to a GitHub repo.

There are four primary methods chained together to perform these task(s).

* `fetch_data()`: uses Oauth2 credentials to authenticate with Google
    and copy data from a spreadsheet into a Python dict.
  
* `transform_data()`: passes each item in the Python dict through a filter,
    providing an opportunity to validate, transform, remove fields, etc.
  
* `make_json()`: converts the Python dict into a JSON array, and provides
    an opportunity to store a local copy of the JSON (useful for testing).
  
* `commit_json()`: authenticates with GitHub and commits a JSON file
    to the identified repository if data has changed since last update.

Running `update_schedule()` will execute these four methods in succession.
It can also be run from the command line: `python update_mozfest_schedule.py`

### Requirements
To install the requirements for this script, create a Python virtual
environment using `virtualenv` and `virtualenvwrapper` (see [this guide](http://www.silverwareconsulting.com/index.cfm/2012/7/24/Getting-Started-with-virtualenv-and-virtualenvwrapper-in-Python)
if you're unfamiliar). Then `pip install -r requirements.txt`.

### Authentication
For authentication to work, you must generate Google and GitHub credentials.
These should be stored as environment variables, and *not* committed to the
repository along with this.

* **GitHub:** Follow these instructions to create a person API token:
    https://github.com/blog/1509-personal-api-tokens
    
    You should generate the token from an account that has write access
    to the repository where you want to store the JSON. It should be stored
    in an environment variable called `GITHUB_TOKEN`.
    
* **Google:** Follow these instructions to create *a service account*:
    https://developers.google.com/console/help/new/?hl=en_US#credentials-access-security-and-identity
    
    Creating a service account will generate a special client email address
    and a new private key. The client email address should be stored in
    an environment variable called `GOOGLE_API_CLIENT_EMAIL`. The private key
    should be stored in an environment variable called `GOOGLE_API_PRIVATE_KEY`.

### Basic project settings
You should change the values in `GITHUB_CONFIG` according to your project.

* **REPO_OWNER:** a string representing the GitHub username of the account
    that owns the repository you want to commit to.
    
* **REPO_NAME:** a string representing the name of the repository to commit to.

* **TARGET_FILE:** a string representing the name of the file you want to create
     or update in the GitHub repository. This can include path information,
     e.g. 'sessions.json', or '_data/sessions.json'

* **TARGET_BRANCHES:** a list representing the branch(es) of the repository you
    want to commit to, e.g. ['gh-pages',] or ['gh-pages','master']

You should also change these values according to your project.

* **GOOGLE_SPREADSHEET_KEY:** a string representing the unique ID of the Google
    spreadsheet storing your data. This can be stored as an environment
    variable called GOOGLE_SPREADSHEET_KEY for extra security, or can simply
    be stored as a string in this script.

* **FETCH_MULTIPLE_WORKSHEETS:** set to `True` if `GOOGLE_SPREADSHEET_KEY` points
    to a document with data in multiple worksheets. The import will retrieve data
    from all worksheets and compile into a single JSON file. _NOTE:_ The import
    will not perform any validation or normalization on column names, so if
    worksheets have varying column names, the resulting JSON objects will have
    varied key names as well.

* **WORKSHEETS_TO_SKIP:** if `FETCH_MULTIPLE_WORKSHEETS` is set to `True`, you
    may define a list of worksheet names to ignore data from.
    e.g. ['Template', '(backup data)']

* **MAKE_LOCAL_JSON:** should likely be set to `False` in production, but can
    be set to `True` for  testing. If set to `True`, the `make_json()` method
    will create a local file containing the JSON for you to inspect.

* **COMMIT_JSON_TO_GITHUB:** should be set to `True` in production. If set
    to `True`, the `commit_json()` method will create or update a JSON file
    in the GitHub repository you specify. Can be set to `False` for testing,
    which will authenticate with GitHub but not create or update any files.

### Data transformations
In many cases, you'll want to apply validation, formatting or transformations
to your data before turning it into JSON, or even derive new keys/values based
on data from specific columns. The `transform_data` method runs every row
of data through an internal `_transform_response_item` function, providing
a place for code that affects each object destined for your JSON. By default,
`_transform_response_item` ensures that each value is translated to a unicode
string, and gives you access to a `_transformed_item` to manipulate.

So if you wanted to, for example, create `day` and `time` values in your JSON
based on concatenated values in a spreadsheet column called `when`, you could
add the code below to `_transform_response_item`:

```
if 'when' in _transformed_item:
    # split values like 'Saturday 7pm' into `day` and `time`
    when = _transformed_item.pop('when', '').split(' ')
    _transformed_item['day'] = when[0]
    _transformed_item['time'] = when[1]
```

You may also set a `skip` flag during the `_transform_response_item` loop
to eliminate a row from the JSON output. For example, if you want to ignore
rows that do not have a numeric value in an `id` column in your spreadsheet:

```
if 'id' in _transformed_item:
    # remove rows without a numeric `id`
    try:
        int(_transformed_item['id'])
    except:
        skip = True
```

### Usage notes
The `fetch_data` method pulls data from the spreadsheet identified in
`GOOGLE_SPREADSHEET_KEY`. If `FETCH_MULTIPLE_WORKSHEETS` is left `False`,
the script assumes that data will be in the spreadsheet's first worksheet:

    datasheet = spreadsheet.get_worksheet(0)
    
If data is *not* in the first worksheet, `spreadsheet.get_worksheet(0)`
can be changed to identify the worksheet by title:

    datasheet = spreadsheet.worksheet("SHEET NAME")
