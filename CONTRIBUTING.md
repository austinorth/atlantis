# Topics
* [Reporting Issues](#reporting-issues)
* [Reporting Security Issues](#reporting-security-issues)
* [Updating The Website](#updating-the-website)
* [Developing](#developing)
* [Releasing](#creating-a-new-release)

# Reporting Issues
* When reporting issues, please include the output of `atlantis version`.
* Also include the steps required to reproduce the problem if possible and applicable. This information will help us review and fix your issue faster.
* When sending lengthy log-files, consider posting them as a gist (https://gist.github.com). Don't forget to remove sensitive data from your logfiles before posting (you can replace those parts with "REDACTED").

# Reporting Security Issues
We take security issues seriously. Please email us directly at security [at] runatlantis.io instead of opening an issue.

# Updating The Website
* To view the generated website locally, run `yarn website:dev` and then
open your browser to http://localhost:8080.
* The website will be regenerated when your pull request is merged to master.


# Developing

## Running Atlantis Locally
Get the source code:
```
go get github.com/runatlantis/atlantis
```
This will clone Atlantis into `$GOPATH/src/github.com/runatlantis/atlantis` (where `$GOPATH` defaults to `~/go`).

Go to that directory:
```
cd $GOPATH/src/github.com/runatlantis/atlantis
```

Compile Atlantis:
```
go install
```

Run Atlantis:
```
atlantis server --gh-user <your username> --gh-token <your token> --repo-whitelist <your repo> --gh-webhook-secret <your webhook secret> --log-level debug
```
If you get an error like `command not found: atlantis`, ensure that `$GOPATH/bin` is in your `$PATH`.

Running Tests Locally:

`make test`. If you want to run the integration tests that actually run real `terraform` commands, run `make test-all`.

Running Tests In Docker:
```
docker run --rm -v $(pwd):/go/src/github.com/runatlantis/atlantis -w /go/src/github.com/runatlantis/atlantis runatlantis/testing-env make test
```

## Calling Your Local Atlantis From GitHub
- Create a test terraform repository in your GitHub.
- Create a personal access token for Atlantis. See [Create a GitHub token](https://github.com/runatlantis/atlantis#create-a-github-token).
- Start Atlantis in server mode using that token:
```
atlantis server --gh-user <your username> --gh-token <your token> --repo-whitelist <your repo> --gh-webhook-secret <your webhook secret> --log-level debug
```
- Download ngrok from https://ngrok.com/download. This will enable you to expose Atlantis running on your laptop to the internet so GitHub can call it.
- When you've downloaded and extracted ngrok, run it on port `4141`:
```
ngrok http 4141
```
- Create a Webhook in your repo and use the `https` url that `ngrok` printed out after running `ngrok http 4141`. Be sure to append `/events` so your webhook url looks something like `https://efce3bcd.ngrok.io/events`. See [Add GitHub Webhook](https://github.com/runatlantis/atlantis#add-github-webhook).
- Create a pull request and type `atlantis help`. You should see the request in the `ngrok` and Atlantis logs and you should also see Atlantis comment back.

## Code Style
### Logging
- `ctx.Log` should be available in most methods. If not, pass it down.
- levels:
    - debug is for developers of atlantis
    - info is for users (expected that people run on info level)
    - warn is for something that might be a problem but we're not sure
    - error is for something that's definitely a problem
- **ALWAYS** logs should be all lowercase (when printed, the first letter of each line will be automatically capitalized)
- **ALWAYS** quote any string variables using %q in the fmt string, ex. `ctx.Log.Info("cleaning clone dir %q", dir)` => `Cleaning clone directory "/tmp/atlantis/lkysow/atlantis-terraform-test/3"`
- **NEVER** use colons "`:`" in a log since that's used to separate error descriptions and causes
  - if you need to have a break in your log, either use `-` or `,` ex. `failed to clean directory, continuing regardless`

### Errors
- **ALWAYS** use lowercase unless the word requires it
- **ALWAYS** use `errors.Wrap(err, "additional context...")"` instead of `fmt.Errorf("additional context: %s", err)`
because it is less likely to result in mistakes and gives us the ability to trace call stacks
- **NEVER** use the words "error occurred when...", or "failed to..." or "unable to...", etc. Instead, describe what was occurring at
time of the error, ex. "cloning repository", "creating AWS session". This will prevent errors from looking like
```
Error setting up workspace: failed to run git clone: could find git
```

and will instead look like
```
Error: setting up workspace: running git clone: no executable "git"
```
This is easier to read and more consistent

### Testing
- place tests under `{package under test}_test` to enforce testing the external interfaces
- if you need to test internally i.e. access non-exported stuff, call the file `{file under test}_internal_test.go`
- use our testing utility for easier-to-read assertions: `import . "github.com/runatlantis/atlantis/testing"` and then use `Assert()`, `Equals()` and `Ok()`
- don't try to describe the whole test by its function name. Instead use `t.Log` statements:
```go
// don't do this
func TestLockingWhenThereIsAnExistingLockForNewEnv(t *testing.T) {
    ...

// do this
func TestLockingExisting(t *testing.T) {
    	t.Log("if there is an existing lock, lock should...")
        ...
       	t.Log("...succeed if the new project has a different path") {
             // optionally wrap in a block so it's easier to read
       }
```
- each test should have a `t.Log` that describes what the current state is and what should happen (like a behavioural test)

# Creating a New Release
1. Update version number in
    1. `main.go`
1. Update `CHANGELOG.md` with latest release number and information (this URL might be useful: https://github.com/runatlantis/atlantis/compare/v0.3.5...master)
1. Create a pull request and merge to master
1. Check out master and fetch latest
1. Run `make release`
1. Go to https://github.com/runatlantis/atlantis/releases and click "Draft a new release"
    1. Prefix version with `v`
    1. The title of the release is the same as the tag (ex. v0.2.2)
    1. Fill in description by copying from the CHANGELOG just without the Downloads section
    1. Drag in binaries made with `make release`
1. Re-run master branch build to ensure tag gets pushed to Docker hub: https://hub.docker.com/r/runatlantis/atlantis/tags/
