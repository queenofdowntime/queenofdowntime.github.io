---
layout: post
title: "More Test-Driven Development in Python"
permalink: /resources/tutorials/crud-py
---

To complete this tutorial, you will need to have the following installed:
- Python
- Pip
- Git
- A text editor
- A terminal or command prompt (we will be working from the terminal a lot, if
	you are not comfortable using yours, you may want to complete a [Command Line Crash Course](https://learnpythonthehardway.org/python3/appendixa.html)
	before you continue).

Think you are missing something? [Check/Install here](https://github.com/fouralarmfire/square-one/blob/master/tutorials/tdd-setup.md).

## quick jump
<ol start="0">
<li><a href="#what-is-an-api">What is an API?</a></li>
<li><a href="#part-1-project-setup">Project setup</a></li>
<li><a href="#part-2-the-first-test">The first test</a></li>
<li><a href="#part-3-the-first-green-test">The first <i>green</i> test</a></li>
<li><a href="#part-4-adding-a-filescreate-endpoint">Create</a></li>
<li><a href="#part-5-adding-a-filesread-endpoint">Read</a></li>
<li><a href="#part-6-adding-a-filesupdate-endpoint">Update</a></li>
<li><a href="#part-7-adding-a-filesdelete-endpoint">Delete</a></li>
<li><a href="#bonus-round">Bonus round!</a></li>
</ol>

## Task:
Using [TDD](https://www.agilealliance.org/glossary/tdd), write **just the back-end** for a simple web API with which you can:

  - Create a text file with some contents stored in a given path.
  - Read the contents of a text file under the given path.
  - Update the contents of a text file.
  - Delete the file that is stored under a given path.

_New to TDD? Start with the Intro Guide [here](https://queenofdowntime.com/resources/tutorials/fizzbuzz-py)._

Example of how our webservice should end up working using [`curl`](https://www.lifewire.com/example-uses-of-the-linux-curl-command-4084144):

```sh
# in one terminal window, start the webservice and set
# the directory in which we want to store the files created by the api
$ python api.py $HOME/crud-files

# in another terminal window

# POST (ie. send) some data the API's 'create' endpoint (ie. URL)
# and then use the command line to see that the webservice has created
# a file based on that data
$ curl -X POST localhost:5000/files/create --data '{"name":"test-file","contents":"hello"}'
Entry 'test-file' created at '~/crud-files'.
$ ls $HOME/crud-files/
test-file
$ cat $HOME/crud-files/test-file
hello

# call that new file's 'read' endpoint (URL) to see its contents
$ curl localhost:5000/files/read/test-file
hello

# PUT (ie. send) some new data to the API's 'update' endpoint (ie. URL)
# for that file and then use the command line to see that the webservice
# has updated the file with the new data
$ curl -X PUT localhost:5000/files/update/test-file --data '{"contents":"goodbye"}'
Entry 'test-file' in '~/crud-files' updated.
$ cat $HOME/crud-files/test-file
goodbye

# call the API's `delete` endpoint (URL) for that file and then use the
# command line to verify that the file has been deleted
$ curl localhost:5000/files/delete/test-file
Entry 'test-file' deleted from '~/crud-files'.
$ ls $HOME/crud-files/

```

## What is an API?

I do intend to write a nice summary here, but until I get around to that please enjoy these links to summaries which others have written!

- [What is CRUD?](https://www.codecademy.com/articles/what-is-crud) _codecademy_
- [What is REST](https://www.codecademy.com/articles/what-is-rest) _codecademy_
- [Back End Architecture](https://www.codecademy.com/articles/back-end-architecture) _codecademy_
- [What exactly is an API?](https://medium.com/@perrysetgo/what-exactly-is-an-api-69f36968a41f)
- [What is an API? In English, please.](https://www.freecodecamp.org/news/what-is-an-api-in-english-please-b880a3214a82/)
- [What is an API?](https://www.mulesoft.com/resources/api/what-is-an-api)


## Part 1: Project setup

**Steps**:

1. Open Terminal (or iTerm or whatever else you like to get a command prompt) and create a new directory.
    Then change into that directory and initialise a new git repository _(New to Git? See [this guide](/resources/tutorials/git).)_:

	```sh
	# note: all text within and including '< >' are placeholders.
	# replace with your own details
	cd ~
	mkdir -p workspace/crud-python
	cd workspace/crud-python
	git init
	git remote add origin <URL OF YOUR REPO ONLINE>
	echo "# CRUD TDD in Python" > README.md
	```

1. We need to ensure that when others wish to use or work on our code, or when we want to clone it to another machine,
	all the _things_ needed to make it work are encapsulated in this repo. This includes any external packages we may use to
	build our API (aka dependencies), and the version of Python we want to use.

	So for this we are going to use [`pipenv`](https://docs.pipenv.org/en/latest/). Their description says
	
	> Pipenv is primarily meant to provide users and developers of applications with an easy method to setup a working environment.

	which is exactly what we want. (If you already have a preferred dependency management tool, go ahead and set up as you normally would,
	you can skip the next couple of steps.)

   Follow their [installation section](https://docs.pipenv.org/en/latest/install/#installing-pipenv) to install `pipenv` on your machine.

1. When it is installed we can run `pipenv --help` to both check that it is working and see what options are available to us.

1. Since Python3 is the future, we will set up our project to use it specifically:

    - First ensure that Python3 is installed on your machine: `python3 --version`. If that returns an error, follow the relevant install
      section [here](https://realpython.com/installing-python/).

    - Next, run: `pipenv --python 3.7`. It should eventually print a colourful success line. You will notice that it has added a `Pipfile` to your repo.

    - Finally, run `pipenv lock` to create a `Pipfile.lock`. The lock file records the precise versions of dependencies in our project which ensures
      builds are [deterministic](https://en.wikipedia.org/wiki/Deterministic_system).

1. Run `pipenv shell`. You should see your terminal prompt change. Run `python --version` (deliberately leaving out the 3), and you should see that the version
    in this environment is exactly the one you asked for. Exit the shell by typing `exit`. Cool, now we are ready to get to work. 

1. First commit this setup and push to your online repo. If you like you can add notes to your README.md as you go to document what you have learned.

	```sh
	git add Pipfile Pipfile.lock README.md
	git commit -m "initial: setup and readme.md"
	git push -u origin master
	```

## Part 2: The first test

**Steps**:
1. We are going to use [`pytest`](https://docs.pytest.org/en/latest/index.html) to write and run our tests. First we need to install
    it in our environment as a dependency. Run the following in your terminal:
    
    ```sh
    pipenv install pytest
    ```

    You will have noticed that your `Pipfile` and `Pipfile.lock` have been updated.

1. To run tests in our environment, we just execute this command in the terminal:

    ```sh
    pipenv run pytest
    ```
    
    Right now it says `no tests ran` which makes sense. Let's write one.

1. Create a new file called `api_test.py`.

1. Open the project in your text editor, and write the following to it:

	```py
	class TestAPI():
	    def test_index(self):
		    return
	```

	Not a whole hell of a lot happening yet, but we can leave it like this to ensure our test runner is working properly.
	Based on the name of that function, we can tell that it will end up testing our API's home or index; the route at `/`.

	Let's read what we've go so far. (For people who are not new to Python, please bear with me.)
	The first line defines our test `class`. This has to begin with `Test`, but the rest is for our benefit.
	It is usually good to pick the name of the element or package you want to test.
	(If you are unfamiliar with Classes and Object Oriented Programming, in Python or otherwise, take a few minutes
	to [read this](https://realpython.com/python3-object-oriented-programming/).)

	After that we define the function which is going to test the first bit of our code.
	Right now our function doesn't test anything and just returns nothing.
	Again we could have called this function anything, but the descriptive name is for our benefit.
	The only rule is that it has to start with `test_` and take `self` (as in the class itself)
	as an argument. This means the test functions will be able to assign and retrieve class-wide attributes.
	We will see what this means in practice in a minute.

1. Run the test again (`pipenv run pytest` in the terminal), and you should see it say `1 passed` in green. 
	Of course we are not actually testing anything: very little can go wrong with returning nothing.

1. So let's write our first real test. We are testing an API, so each test will hit a different endpoint of that API and check the
	result is what we expect.

	To call our API from our test we can use the [`requests` package](https://3.python-requests.org/).

	Alter your `test_index` function to look like this:

	```py
	def test_index(self):
	    r = requests.get("http://127.0.0.7:5000/")
	    assert r.status_code == 200
	```

	Here we are making a `GET` request to an index endpoint which will be running on our machine's local server (`127.0.0.1`) at port `5000`.
	Ports are the "doors" through which services provided by servers can be accessed, so anything trying to communicate with our API will do so via port 5000 (more on why 5000 later)

	Finally, since `requests` is a new package, we need to import it at the top of our file so that our tests can use it:

	```py
	import requests
	```

1. Run the test: `pipenv run pyest`.
	It fails. Have we forgotten something? The clue is in the failure: `No module named 'requests'`.
	We need to install `requests` in our environment (using the command line):
	
	```sh
	pipenv install requests
	```

1. When we run the tests again... the failure is very messy but essentially `Failed to establish a new connection` means that there isn't a server to connect to.

## Part 3: The first _green_ test

**Steps**:

1.  We are going to use [`Flask`](http://flask.pocoo.org/docs/1.0/) to build our API.
    Flask is a very simple barebones web framework. Flask's default port
    is 5000... so that explains why in our test we are going to use it to talk to our API.

    Install flask in our environment: `pipenv install flask`.

1. Create a new file called `api.py` and open it in a text editor.

1. Paste in the following snippet:

    ```py
    from flask import Flask

    app = Flask(__name__)

    @app.route('/')
    def index():
	    return "welcome home"

    app.run(debug=True)
    ```

    What have we done here?

    On the first line we have imported the Flask class from the flask module.
    The line below that defines our `app` which is an instance of the Flask class.
    (The name of the instance is up to you, but choose something which makes sense).
    `__name__` is a special variable in Python which you can read about [here](https://www.geeksforgeeks.org/__name__-special-variable-python/).

    Next we register our first route on the app, `/` aka home. Immediately after that we define
    what we want to happen when somebody calls that route. This function has to return _something_; flask will
    crash if you don't at least return an empty string. We wont be using `/` for anything for a bit
    so you can return whatever you like here. You could use it to return instructions on how to use
    the service, or just remove the endpoint alltogether. For now it is useful to get us started.

    And finally we have the line which will run our server, with debug mode turned on while we develop.

1. What happens when we run our test? It fails.	This is because our test is expecting a running server,
    but we haven't _actually_ started ours.

    We need to start the server in one terminal window, run the tests in another, then kill the server in the first. This sounds like a pain
    to run manually every time, so we are going to automate. Fortunately, `pytest` has a thing for that.

1. Open your `api_test.py` file again, and add the following section _just above_ your `test_index` method:

    ```py
    @pytest.fixture(autouse=True)
    def start_server(self):
       server = pexpect.spawn("python api.py")
       server.expect('Running on http://127.0.0.1:5000')
       yield server
       server.kill(9)
    ```

    Let's go through line by line.

    First we declare that we want to set up a new [`fixture`](https://docs.pytest.org/en/latest/fixture.html). A fixture is a `pytest`
    mechanism which lets us declare methods or functions which are common to all tests within a suite. In our case, we want our server
    to start, wait to be acted upon, then stop for each and every test we are about to write: they all have a common need. This saves us
    from having to paste that start/act/stop in each test, which would look very messy and go against the DRY (Don't Repeat Yourself) principle.
    We are going to use fixtures to perform common `setup` and then `teardown` steps around all our tests.

    Crucially we include `autouse=True` as a parameter to our fixture. This is so that we don't have to explicitly pass it and call it within the
    tests that we want it to apply to; it is called, and our server booted, automatically before each test. (We will see a non-auto fixture in a moment.)

    The third line, after the name of the fixture function is declared, uses the `pexpect` package to start the python server as a child process in the background.
    "In the background" means that the current process running our tests is not blocked on the server as it starts and runs. If you run `python api.py` on your
    command line, you will notice that after it starts running, it just sits there, and you can't type a further command until you `CTRL-C` (interrupt/stop) the server.
    Obviously we don't want our tests to just get stuck waiting forever like that.

    But we also don't want our tests to start running before the server has actually had a chance to start. Therefore on the following line, we say that we eventually expect
    to see that backgrounded server process print `'Running on ...'`, which if you ran if manually, is the output you would have seen reported when the server was ready.

    We obviously don't want our server to keep running forever (for one thing a subsequent run of the tests would fail because it would complain that a server already exists).
    So on the very last line we kill the background process, specifying a [`SIGNAL`](https://unix.stackexchange.com/a/317496) of `9`; aka the kill code.

    But what about the second to last line, just before the kill? The `yield` keyword is a complex one, and beyond the scope of this tutorial. (You can learn about it [here](https://pythontips.com/2013/09/29/the-python-yield-keyword-explained/).)
    In our fixture, a `yield` lets us put our current method 'on pause' at that line. The test which called our fixture (thanks to `autouse` each test will call it implicitly before it runs)
    then gets started, interacts with the running server and tests what it needs to test. When the test method is done, the fixture picks up again from where it was halted, and proceeds to its next line.

    Using a fixture like this we can start our server, wait for it to be ready, leave it running while we use it, then come back and shut it down when we are done. Cool.
    
1. Before we can use it, we have to remember to put some things at the top of our test file:

    ```py
    import pexpect
    import pytest
    ```
    
    and also add our new package to our Pipenv dependencies: `pipenv install pexpect`.

1. And with that we can finally run: `pipenv run pytest`

    Woohoo our first test is (legitimately) green!

    _**Note**: `pexpect` will not work on Windows, but a similar result can be achieved by combining [`subprocess` + `time.sleep`](https://stackoverflow.com/a/40156864)._

1. Before we commit this to Github, there is a little tidying up we can do.

    Right now the URL for our test server appears twice in our test file. This may not seem like a lot, but since every one of our tests is going to call that same URL, it would be sensible to
    save that to a variable somewhere. Saving commonly used values like this not only keeps things looking neat, it also means that if we were to ever change the value (e.g. go for a different
    port number), we would only have to switch the value in one location, and not spend ages looking for several.

    We can use another fixture for this! Define a new one above our `start_server` fixture called `setup`:

    ```py
    @pytest.fixture(autouse=True)
    def setup(self):
        self.url = 'http://127.0.0.1:5000'
    ```

    Now that we have saved our URL to the class, we can use it in our other fixture as well as our test: replace the old mentions of `http://127.0.0.1:5000` with `self.url`.

    _**Tip**: If your tests ever hang for a while, there is likely something wrong with your `api.py` code._

1. We can now commit this and push to Github.

	```sh
	git add Pipfile Pipfile.lock api.py api_test.py
	git commit -m "add index endpoint"
	git push
	```

[How your project should look](https://github.com/Callisto13/crud-py/tree/608d511e7e08fe7f7ab0ad7f578b65efaea7aa3e) at this stage.

## Part 4: Adding a /files/create endpoint

When our API is done, we want to be able to send information about a file we'd like created to a URL, and see that a file _is_ created.
This is how we write our tests: we perform the steps we would as if the code was already written, and then verify that the thing we
wanted to happen _did_ happen.

So in our first test we want to do the following:
- Make a request to that endpoint
  - With the information we want the API to act on
- Verify that the returned HTTP code shows that the call was a success
- Verify that there is now a file which matches the data we provided

Let's do this one bit at a time.

**Steps**:

1.  Open `api_test.py` in a text editor and paste the following below our previous test:

    ```py
    def test_post_create(self):
	    content_header = {'Content-Type': 'application/json'}
	    data = {'name': 'test-file', 'contents':'hello'}
	    r = requests.post(self.url+"/files/create", headers=content_header, json=data)
	    assert r.status_code == 201
    ```
  
    What have we done here?

    The first line creates a `header` variable which we can use to stipulate that the type of data we are sending
    through the request will be [`json`](https://stackoverflow.com/a/41656941). On the next line we then create that `data` variable with information which is
    organised as... [`json`](https://www.copterlabs.com/json-what-it-is-how-it-works-how-to-use-it/)!

    Next we make our `POST` request to the `/files/create` endpoint, using the `data` and our `content_header`.

    Lastly after we make our request, we want to check that the create was a success, so we match the response `status_code`
    against what we want it to be. By checking a handy site like [HTTPStatusDogs](https://httpstatusdogs.com/), we know that
    successfully creates usually return `201` exit codes.

1. Run your tests: `pipenv run pytest`.

    This should fail with `E       assert 404 == 201`. If we check our [Status Dogs](https://httpstatusdogs.com/), we
    know that `404` means that the URL we asked for was `not found` by the server. Let's go and give it something to find.

1. Open `api.py` in a text editor, and create new route at `/files/create` with its associated function underneath:

    ```py
    @app.route('/files/create')
    def create():
	    return ""
    ```

    Don't make it return anything other than an empty string for now.

1. Run your tests.

    This time we get a 405. Our puppies tell us this means `Method not Allowed`. The default for a new `@app.route` is
    a `GET` request, but we are doing a `POST` here.

    Update your route definition to make sure it does the right thing:

    ```py
    @app.route('/files/create', methods=['POST'])
    ```

1. Run the tests again.

    `200`? We are getting closer. Again the default response code for any successful request is `200`, so we need
    to make it return the thing we actually want.

    Add the correct code to your return:

    ```py
	    return "", 201
    ```

   Now we should be green again.

1.  Let's add a little more user feedback to this experience. It would be nice if, when a file is successfully created,
    a message would appear telling us about it.

    In `api_test.py` add a new line at the bottom of our `test_post_create`:

    ```py
	    assert r.text == "File 'test-file' created."
    ```

    We are checking that when we make the `POST` request to create a file, we receive some text back which we can assert
    matches what we expect.

1. Run the tests and see them fail.

    In order to include the correct file name in the output, our `create` function in `api.py` is going to need to read the data
    posted to it.

    In `api.py` alter your function with the following to read the `json` from the request, and then retrieve the value of the file `name` key:

    ```py
	    data = request.get_json()
	    return "File '{}' created.".format(data['name']), 201
    ```

    We are using a module called `request` (not to be confused with the `requests` package in our test file). This one
    comes from `flask`.

    Edit the top line of your file to import `request` too:

    ```py
    from flask import Flask, request
    ```

    We should be green again. But I am sure you have noticed: despite our code claiming that we have created a file, we have done no such thing.

    Let's expand the test to force ourselves to actually do the work.

1. Back in `api_test.py`, add some more lines to the bottom of our `test_post_create` function:

    ```py
	    file_object = open("test-file", "r")
	    read_content = file_object.read()
	    file_object.close()
	    assert read_content == "hello"
    ```

    Here, after we have made our request, we open the file we named in our request `data`, read it, and check its contents against what we expect.

    If we run the test now it fails with a `FileNotFoundError`.

1. In `api.py` we can change `create()` to use the data from the request to create the desired file with
  the correct contents:

    ```py
    def create():
        data = request.get_json()
        name, contents = data['name'], data['contents']
        f = open(name, "w")
        f.write(contents)
        f.close()
        return "File '{}' created.".format(name), 201
    ```

    This makes the test pass, but we have a problem: if you run `ls` in your project directory, you will
    notice that `test-file` is now there. We don't want to end up with a whole load of junk
    files in our directory every time we run the tests.

    The common way to handle this is to ensure that any artifacts created by tests are deleted at the end of the run.
    Obviously if a test is interrupted _before_ it gets to the cleanup step, then things could still end up lying around
    where we don't want, just waiting to be committed accidentally when we forget.

1. So we are going to ensure that all files created by our tests end up in a temporary directory in an unimportant location which we can
    completely delete at the end. (Don't forget to `rm` that `test-file`.)

    Time for another fixture!

    In `api_test.py` past the following snippet somewhere in our fixtures area, between `setup` and `start_server` is fine:

    ```py
    @pytest.fixture()
    def tmp_dir(self):
        tmp = tempfile.mkdtemp(prefix = "pcrud")
        yield tmp
        shutil.rmtree(tmp)
    ```

    What we have here is pretty similar to our other fixtures. This one does not have `autouse` set because we are actually going to call
    this fixture from within another.

    The first line of the method creates a temporary directory in your operating system's default location for storing temporary things. We have
    prefixed it with `"pcrud"` so that if things go wrong, we can find the created files later.

    Next we `yield` so that the calling test can run, and lastly we erase the entire thing, leaving a clean slate for the next run.

    There are a couple of new modules here so you will need to add them to the top of your test file:

    ```py
    import shutil
    import tempfile
    ```

    They are built into Python, so you don't need to install them into your environment.

    This new fixture will be called from our `setup` fixture. To do this we need to pass it in as an argument, and then we can save the location of
    our new temp directory to the `class` so that all tests can access it:

    ```py
    ...
    def setup(self, tmp_dir):
         self.url = 'http://127.0.0.1:5000'
         self.tmp = tmp_dir
    ```

1.  We are ready for our server to use our new temporary test directory.
    Luckily, Flask has a handy built-in config option, which means that we can stipulate a location for all the files our api interacts with.

    This is passed in as a command line argument, so on the line in our `start_server` fixture where we are booting our server, we can add our temp dir at the end:

    ```py
             server = pexpect.spawn("python api.py "+self.tmp)
    ```

    Then in `api.py` add these two lines **directly above** your index (`/`) route declaration:

    ```py
    app.config['store'] = sys.argv[1]
    store = app.config.get('store')
    ```

    Here we use the `sys` module to grab the first argument from the command line as the server starts, and save it to the app's config object.
    On the next line we save the config value to another varaible, simply because using `store` everywhere (as we are about to) looks a little neater
    than `app.config.get('store')`.

     Don't forget to `import sys` at the top of `api.py`.

    Back in `test_post_create`, we also have to make sure we are opening the correct file to verify the contents after our test posts the data:

    ```py
    file_object = open(self.tmp+"/test-file", "r")
    ```

1. When we run our tests now, they should fail.

    Open `api.py` and change the code so that it knows to use the `store` config option we have set:


    ```py
    ...
    f = open(store+'/'+name, "w") 
    ...
    ```

1. Now our tests should be back to green. Congrats! You can now commit and push to Github:

	```sh
	git add api.py api_test.py
	git commit -m "add create endpoint"
	git push
	```

[How your project should look](https://github.com/Callisto13/crud-py/tree/6bba20c9899b02abae34208ee75a8e1a8a803412) at this stage.


## Part 5: Adding a /files/read endpoint

Now we have created a file at the location we want, let's read it back.

**Steps**:

1.  Open `api_test.py` in a text editor and paste the following below our `test_post_create` test:

    ```py
    def test_get_read(self):
        r = requests.get(self.url+"/files/read/test-file")
        assert r.status_code == 200
    ```

    This is a little simpler than the `create` since we don't need to send json data through: the resource should exist at its own endpoint (URL).

1. Run the test. It should fail with a predictable `404` aka `not found`.

1. Open `api.py` and create the new route:

    ```py
    @app.route('/files/read/<filename>')
    def read(filename):
        return ''
    ```

    This one looks a little different to the others. In order for us to do a `GET` request on `https://blah:1234/files/read/some-random-name`, instead of sending data to
    `.../files/read` as we did with `create`, we need to be able to pass in that filename as a variable. We can then use the same route for all files.

    Once we have defined the variable name with the `< >` syntax, we pass it into the `read` function so that we can use it later.

1. The tests should be green again. But nothing is happening yet in that function, so let's expand our `test_get_read`.

   Right now our tests are trying to read a file which doesn't yet exist (the file we created in `test_post_create` does not exist in the scope of this test; this is a whole new file).
    So we need to do some test prep so that our api has something to work with.

    Slot in the following at the top of `test_get_read` (just before the `requests.get` line):

    ```py
         expected_contents = "contents of the test file"
         file_object = open(self.tmp+'/test-file', "w")
         file_object.write(expected_contents)
         file_object.close()
    ```

    Then at the bottom of `test_get_read` add one more assert:

    ```py
            assert r.text == expected_contents
    ```

1. Now that the test is failing in a useful and reliable way, we can code the solution.

    In `api.py`, fix the `read` function so that it uses the variable passed in via the URL to open the correct file in the store and return the contents:

    ```py
     @app.route('/files/read/<filename>')
     def read(filename):
         f = open(store+'/'+filename, "r")
         contents = f.read()
         f.close()
         return contents
    ```

1. The tests should be all green and we are half-way there! You can commit and push this new functionality to Github:

	```sh
	git add api.py api_test.py
	git commit -m "add read endpoint"
	git push
	```

[How your project should look](https://github.com/Callisto13/crud-py/tree/d224e414e06cc698d1fc5ae8954d5dafb3cb6557) at this stage.

## Part 6: Adding a /files/update endpoint

The next endpoint we need is one which can update an already existing file in the store.

**Steps**:

1. In `api_test.py` add a new test underneath the last:

    ```py
     def test_put_update(self):
         content_header = {'Content-Type': 'application/json'}
         data = {'contents':'new shiny updated contents'}
         r = requests.put(self.url+"/files/update/test-file", headers=content_header, json=data)
         assert r.status_code == 200
         assert r.text == "File 'test-file' updated."
    ```

    There is a bit of a mix between the previous two tests here: we need to send the data of the new contents through as json, but the file resource/endpoint already exists, which lets us use the variable
    on that route.

    The method we use to update a resource is `PUT`.

1.  The test should fail of course, so open `api.py` and create the `update` route, not forgetting to return some nice user feedback:

    ```py
    @app.route('/files/update/<filename>', methods=['PUT'])
    def update(filename):
        return 'File '{}' updated.'.format(filename)
    ```

1. Similar to `test_get_read` we want to set up a file for the api to update, and then verify that it was updated with what we asked for.

    At the top of `test_put_update` create a new file:

    ```py
         f = open(self.tmp+'/test-file', "w")
         f.write('boring old contents')
         f.close()
    ```

    Then create a variable for the new contents and use it in the `data` for the request, replacing what you pasted in earlier:

    ```py
         expected_new_contents = 'new shiny updated contents'
	     ...
         data = {'contents': expected_new_contents}
	     ...
    ```

    Finally, we will assert that the contents of the file match the data which we sent through the api request.
    To do this we have to read that file again:

    ```py
         file_object = open(self.tmp+"/test-file", "r")
         read_content = file_object.read()
         file_object.close()
         assert read_content == expected_new_contents
    ```

1. Now that the test is failing with `AssertionError: assert 'boring old contents' == 'new shiny updated contents'` we can go implement the `update` function in `api.py`:

    ```py
    @app.route('/files/update/<filename>', methods=['PUT'])
    def update(filename):
        data = request.get_json()
        contents = data['contents']
        f = open(store+'/'+filename, "w")
        f.write(contents)
        f.close()
        return "File '{}' updated.".format(filename)
    ```

    [How your project should look](https://github.com/Callisto13/crud-py/tree/7ce552853e42c21678fbe90db1d110d0f4cb72ba) at this stage.

1. The tests are green again, but before we commit we should probably take a minute to refactor our tests. Reading a writing files are happening in a couple of places, so it would
    be nice if they were put into some nice reusable functions somewhere.

1. Once you have done that, and all tests are passing again, you can commit and push to Github:

	```sh
	git add api.py api_test.py
	git commit -m "add update endpoint"
	git push
	```

[How your project should look](https://github.com/Callisto13/crud-py/tree/3d13e4be46ae4f1c3be460d5cf6548bc6708df6c) at this stage.


## Part 7: Adding a /files/delete endpoint

The final part of our CRUD api is Delete!

**Steps**:

1. In `api_test.py` add another test under the last:

    ```py
     def test_delete_delete(self):
         write_file(self.tmp+'/test-file', 'goodbye')
         r = requests.delete(self.url+"/files/delete/test-file")
         assert r.status_code == 200
         assert r.text == "File 'test-file' deleted."
         assert os.path.exists(self.tmp+'/test-file') == False
    ```

    As before, we are prepping a file for the api to delete. We are using the `DELETE` http method call, which we expect to return a `200` code, we check for some nice user feedback, and we verify that
    the file no longer exists.

    _(A `DELETE` can return a `204`, but this would remove the content of the response, meaning we lose our nice user feedback.)_

    The test name is a little weird, I will grant you. But I settled on this naming convention at the beginning so we are stuck with it.

    We are using a new module here, so don't forget to `import os.path` at the top of `api_test.py`.

1. This fails with the expected `404`, so we can return to `api.py` and make it go a little further:

    ```py
    @app.route('/files/delete/<filename>', methods=['DELETE'])
    def delete(filename):
       return "File '{}' deleted from '{}'.".format(filename, store)
    ```

1. Run the tests again, and the next failure we get is the most hideous mess. The last line has been reached, and it is complaining that a file we said shouldn't be there, is still there.
    Let's update `api.py` to do the right thing:

    ```py
    import os

    ...

    @app.route('/files/delete/<filename>', methods=['DELETE'])
    def delete(filename):
        os.remove(store+'/'+filename)
        return "File '{}' deleted from '{}'.".format(filename, store)
    ```

    And that's it! You have successfully built a CRUD api using TDD!

1. When you are happy with how everything looks, and all your tests are passing, you can commit and push to Github:

	```sh
	git add api.py api_test.py
	git commit -m "add delete endpoint"
	git push
	```

[How your project should look](https://github.com/Callisto13/crud-py/tree/1344972766204de51307eae16a16211af8249fe9) at this stage.



## Bonus Round!

There are many more things you can do to expand your CRUD api, all with TDD ofc!

- Why not add a `/files/read` endpoint which lists all files currently in the store? (What happens if there are no files?) _[example](https://github.com/Callisto13/crud-py/tree/773b5d52fe6fa03f2b0c8967ac4beff88763b7b7)_
- Should the `create` function check that the file does not exist before it over-writes a file?
- Should the `update` function also check that it is updating an existing files?
- Likewise the `delete` function: maybe it should fail if it cannot find a file to delete?
- What if we wanted to create a file inside another folder _inside_ the store? `/files/create/like/this`?
- What should happen if the `store` directory does not exist? Should the API create one? Should it fail?
- Are there any other 'edge-cases' (unforeseen ways the webservice could be used) which you could guard against?
- This tutorial only focuses on the back-end, what if we threw on a front-end? (For that I will probably write a new tutorial, but please give it a try while I procrastinate!)

&nbsp;

<sup>_Mistakes? Corrections? Suggestions?_ <a href="https://github.com/queenofdowntime/queenofdowntime.github.io/tree/master/tutorials/crud-py.md"><svg class="svg-icon"><use xlink:href="/assets/minima-social-icons.svg#github"></use></svg></a></sup>

<sup>_Is something unclear? Do you need help?_ <a href="https://github.com/queenofdowntime/queenofdowntime.github.io/issues/new"><svg class="svg-icon"><use xlink:href="/assets/minima-social-icons.svg#github"></use></svg></a></sup>
