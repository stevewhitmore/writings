# Python Powered Pipelines? Preposterous!

![Python woman image](https://res.cloudinary.com/practicaldev/image/fetch/s--WIpmUB8k--/c_imagga_scale,f_auto,fl_progressive,h_420,q_auto,w_1000/https://dev-to-uploads.s3.amazonaws.com/uploads/articles/tr2e4wrn2ynz2d9vbdd3.jpg)

<p class="tagline">devops, python, angular, productivity<br>
Crossposted to <a href="https://dev.to/stevewhitmore/python-powered-pipelines-preposterous-4316">Dev.to</a>.</p>

No, not preposterous. Powerful.

DevOps is all about rapid delivery. Using Python to make your pipelines "smart" can help you achieve that goal.

Let's say you have a suite of Angular libraries your team created to use for their applications. A robust pipeline for this project will include the following:

- validation jobs that make sure
  - there's no "fit" or "fdescribe" to narrow down unit tests
  - there's no "dist" or ".angular" folders present
  - eslint passes
- tests
  - unit tests
  - integration tests
- security scans
  - npm audit
  - SAST scans (SonarQube, Fortify, etc)
- publication
  - snapshots
  - release candidates
  - releases

At an enterprise level, you could be dealing with upwards of 20 libraries. You *could* have a job for each library. Speaking from experience, having jobs for each will quickly turn your pipeline into a big nasty pile of spaghetti. Nobody wants to deal with a big nasty pile of spaghetti.

Worse still, the time it'll take for your pipelines to run will slow things down to an agonizing crawl and have your team pulling their hair out in frustration every time they push a change up.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ixh82g3810q8m8phyyr1.jpg)

*Or* you can have an easy to read and maintain Python script that will figure out which libraries actually changed, then run the jobs for those libraries.

Let's use a mock scenario to illustrate my point. For the sake of brevity we'll just have our pipeline:

- Run eslint
- Run unit tests
- Publish snapshots off of our feature branches

[Here's a project](https://github.com/stevewhitmore/python-powered-pipelines) that demonstrates this using both GitHub Actions and GitLab CI/CD. I'm including both because I think most people use GitHub Actions but I know many companies also use GitLab for their enterprise applications (plus I'm way more familiar with GitLab CI/CD).

The major points I'll be going over will be:

1. Setting up a runner image
2. Setting up your CI file
  - GitHub Actions
  - GitLab CI/CD
3. Handling secrets
  - Creating an auth token for npmjs.org
  - Creating Actions secrets in GitHub
  - Creating masked environment variables in GitLab
4. Pythonizing your pipeline

## Setting up a runner image

There are a boat-load of Docker images to choose from for using as your pipeline runner image. Personally, I like having total control over my pipelines and prefer to use my own image. I like Alpine because it's tiny compared to the more popular Ubuntu. Tiny is good because it loads faster and there's a smaller attack surface. 

Based on the listed requirements above, our image will need to support Python, the packages our pipeline script will be using, nodejs, and a browser for our unit tests.

Here's a good example of an image that will serve our needs:

```dockerfile
FROM alpine:latest

RUN apk add --no-cache --update python3-dev gcc libc-dev libffi-dev git && \
    ln -sf /usr/share/zoneinfo/America/Chicago /etc/localtime && \
    ln -sf python3 /usr/bin/python && \
    ln -sf pip3 /usr/bin/pip

COPY ./scripts/requirements.txt /tmp

RUN python -m ensurepip && \
    python -m pip install --no-cache --upgrade -r /tmp/requirements.txt

RUN apk add --update --repository http://dl-cdn.alpinelinux.org/alpine/v3.16/main nodejs=16.17.1-r0 npm && \
    apk add --no-cache chromium --repository http://dl-cdn.alpinelinux.org/alpine/v3.16/community

ENV CHROME_BIN=/usr/bin/chromium-browser CHROME_PATH=/usr/lib/chromium
```

Super duper. Assuming you know your way around Docker we can publish our image to whatever registry we use. If you're not familiar with Docker, don't fret. Their [documentation](https://docs.docker.com/) is outstanding and well worth spending time reading through.

## Setting up your CI file

### GitHub Actions

As far as I can tell GitHub doesn't support using a custom pipeline runner image directly. You'll need to wrap a natively supported image around it and set your image as the "container". Kinda icky but it will still serve you.

> **Disclaimer:** I'm still learning my way around GitHub Actions so the below yml file is far from perfect. It *works* but I'd love to hear about ways it could be improved/optimized.

<p class="caption">.github/workflows/ci.yml</p>

```yml
name: CI
on: push

jobs:
  CI:
    runs-on: ubuntu-latest
    container: stevewhitmore/nodejs-python

    steps:
      - uses: actions/checkout@v3

      - name: Cache node modules
        id: cache-npm
        uses: actions/cache@v3
        env:
          cache-name: cache-node-modules
        with:
          path: ~/.npm
          key: ${{ runner.os }}-build-${{ env.cache-name }}-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-build-${{ env.cache-name }}-
            ${{ runner.os }}-build-
            ${{ runner.os }}-

      - name: Allow me to run my script
        run: git config --global safe.directory '*'

      - name: Lint
        run: python scripts/ci.py eslint

      - name: Unit Tests
        run: python scripts/ci.py unit_tests

      - name: Publish Snapshots
        run: |
          echo "//registry.npmjs.org/:_authToken=${{ secrets.NPM_TOKEN }}" > .npmrc
          python scripts/ci.py publish_snapshots
```

### GitLab CI/CD

<p class="caption">.gitlab-ci.yml</p>

```yml
image: stevewhitmore/nodejs-python

stages:
  - validation
  - test
  - snapshot

cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - node_modules/
    - .npm/

pylint:
  stage: validation
  script:
    - python -m pylint --version
    - PYTHONPATH=${PYTHONPATH}:$(dirname %d) python -m pylint scripts/ci.py
  except:
    - tags

eslint:
  stage: validation
  cache:
    key: ${CI_COMMIT_REF_SLUG}
  script:
    - python scripts/ci.py eslint
  except:
    - tags

unit_tests:
  stage: test
  cache:
    key: ${CI_COMMIT_REF_SLUG}
  script:
    - python scripts/ci.py unit_tests
  except:
    - tags

npm_publish_snapshot:
  stage: snapshot
  cache:
    key: ${CI_COMMIT_REF_SLUG}
  script:
    - echo "//registry.npmjs.org/:_authToken=${NPM_TOKEN}" > .npmrc
    - python scripts/ci.py publish_snapshots
  except:
    - main
    - tags
```

## Handling secrets

NPM needs to know where your packages (libraries) will be registered and there needs to be some kind of authentication. It gets this information from the `.npmrc` file. Assuming you're publishing to npmjs.org, your pipeline's `.npmrc` file will look pretty similar. 

> **Note:** This `.npmrc` file is created and only exists during the lifecycle of the pipeline job. DON'T use the `.npmrc` file from your personal workstation! 

The `NPM_TOKEN` is an environment variable we'll pass from the project's settings. **Be sure to mask this variable** or you'll be inviting the world to publish npm packages on your bahalf. GitHub does this automatically but GitLab requires an additional step.

### Creating an auth token for npmjs.org

1. Sign into npmjs.org and click on your username on the far right. Select "Access Tokens"

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/u8yno2wdoy1e0mfsxw3t.png)

2. Click "Generate New Token" on the far right

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/9cdcr12ud8j6d42ce063.png)

3. Name your token and select the "Automation" option. This is the ideal option because it will bypass two-factor authentication (which you absolutely should have set up).

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/bu8p7kr7osmsp4qlmg3e.png)

4. Copy the token to a file so you don't lose it.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/m59n7raxj1rjydpdm212.png)
*No, you can't use this token. It has been deleted* 😉

### Creating Actions secrets in GitHub

1. Go to the project Settings > Secrets > Actions. Click "New repository secret"

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/d9t8ptpyz4obo5p2rd29.png)

2. Fill out the "Name" and "Secret" keys with NPM_TOKEN and whatever your auth token is, then click "Add secret"

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/maq8q6qgm7ecj4638is1.png)

### Creating masked environment variables in GitLab

1. Go to the project Settings > CI/CD

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/jfykl843lh7goyg8927q.png)

2. Expand the "Variables" section and click "Add Variable"

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/vrd3pd723j4j5xo8hou1.png)

3. Add your auth token to your project. Note the **Masked** box is checked.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/drtqooao9vqbrsxse28i.png)

## Pythonizing your pipeline

Now for the fun part. Let the Pythonization commence!

Create a folder named "scripts" at the root of your project. In that folder, create a file `ci.py`. 

There was an argument passed in for each Python script call in our CI file. Those arguments in turn are intended to trigger a specific function that will live in the script file.

For example, the `unit_tests` job has the following line:

```yml
python scripts/ci.py unit_tests
```
We're passing `unit_tests` to the `ci.py` file.

*scripts/ci.py*
```python
import sys

# ...

def unit_tests():
    """Runs unit tests on libraries with changes"""
    npm_command("test")

locals()[sys.argv[1]]()
```

Let's assume `npm_command()` handles whatever npm command you pass in (shocking, I know). It would look something like this:

```python
def npm_command(command):
    """Runs npm commands depending on input"""
    npm_install()
    diffs = get_diffs()

    for library in diffs:
        subprocess.check_call(f"npm run {command}-{library}", shell=True)
```

That `get_diffs()` function uses the GitPython package to compare the changes on your branch with the default origin branch (main). It finds all the git diffs, plucks out the library name, and returns a set of library names to avoid duplication.

```python
def get_diffs():
    """Gets the git diffs to determine which libraries to run operations on"""
    path = os.getcwd()
    repo = Repo(path)
    repo.remotes.origin.fetch()
    diffs = str(repo.git.diff('origin/main', name_only=True)).splitlines()

    updated_libraries = []

    for diff in diffs:
        if diff.startswith("projects"):
            path_parts = diff.split("/")
            updated_libraries.append(path_parts[1])

    return set(updated_libraries)
```

Great, but what about something a little more complex, like publishing snapshots? NPM doesn't allow for duplicate version numbers, so how would we handle that?

Let's take another look at that `npm_publish_snapshot` job.

```yml
python scripts/ci.py publish_snapshots
```

So it'll call the `publish_snapshots()` function in our script.

```python
def publish_snapshots():
    """Publishes npm snapshots on libraries with changes"""
    npm_command("publish snapshots")
```

Not super helpful so far. Let's take another look at that `npm_command()` function.

```python
def npm_command(command):
    """Runs npm commands depending on input"""
    npm_install()
    diffs = get_diffs()

    for library in diffs:
        if command == "publish snapshots":
            handle_snapshot_publication(library)
        else:
            subprocess.check_call(f"npm run {command}-{library}", shell=True)
```

Now there's an if/else block in our loop. The `handle_snapshot_publication()` function should append a unique snapshot version to the changed library, build, then publish it.

```python
def handle_snapshot_publication(library):
    """Updates version with snapshot, builds, and publishes snapshot"""
    package_json_path = f"./projects/{library}/package.json"
    with open(package_json_path, "r", encoding="UTF-8") as package_json:
        contents = json.load(package_json)

    version = contents["version"]
    timestamp = datetime.now().strftime("%Y%m%d-%H%M%S")
    is_snapshot_version = re.match("\\s*([\\d.]+)-SNAPSHOT-([\\d-]+)", version)

    if is_snapshot_version:
        version = version.split("-")[0]

    contents["version"] = f"{version}-SNAPSHOT-{timestamp}"
    with open(package_json_path, "w", encoding="UTF-8") as package_json:
        package_json.write(json.dumps(contents, indent=2))

    subprocess.check_call(f"npm run build-{library}", shell=True)
    subprocess.check_call(f"npm publish --access=public ./dist/{library}", shell=True)
```

The above function reads the changed library's `package.json` file, parses out the version number, and replaces it with the version number plus a timestamp. So a version `1.2.3` becomes version `1.2.3-SNAPSHOT-{year/month/day-hour/minute/second}` (e.g. `1.2.3-SNAPSHOT-20221110-075530`). It also has a check in there `is_snapshot_version` in case you're rerunning a job. This will avoid funky versions from being generated, like `1.2.3-SNAPSHOT-{timestamp}-SNAPSHOT-{timestamp}`.

Let's clean that up a little bit to be more singly-minded.

```python
def append_snapshot_version(library):
    """Appends "-SNAPSHOT-" plus timestamp (down to the second) to library version"""
    package_json_path = f"./projects/{library}/package.json"
    with open(package_json_path, "r", encoding="UTF-8") as package_json:
        contents = json.load(package_json)

    version = contents["version"]
    timestamp = datetime.now().strftime("%Y%m%d-%H%M%S")
    is_snapshot_version = re.match("\\s*([\\d.]+)-SNAPSHOT-([\\d-]+)", version)

    if is_snapshot_version:
        version = version.split("-")[0]

    contents["version"] = f"{version}-SNAPSHOT-{timestamp}"
    with open(package_json_path, "w", encoding="UTF-8") as package_json:
        package_json.write(json.dumps(contents, indent=2))

def handle_snapshot_publication(library):
    """Updates version with snapshot, builds, and publishes snapshot"""
    append_snapshot_version(library)
    subprocess.check_call(f"npm run build-{library}", shell=True)
    subprocess.check_call(f"npm publish --access=public ./dist/{library}", shell=True)
```

By now you should be seeing the pattern. You can see from the job outputs that the pipeline is only running the jobs on the changed libraries:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/c7krn076ly78b96sighw.png)

<p class="caption">GitHub</p>

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/wl8rltfk462lqh1e9qgg.png)

<p class="caption">GitLab</p>

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/o0hdnv4ki3frnvl4gm4j.png)

Enjoy a less painful pipeline with the power of Python! 🦸‍♂️


