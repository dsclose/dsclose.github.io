Installing RVM and Ruby on a new CentOS machine
-----------------------------------------------

Install RVM on the system:

```bash
curl -sSL https://get.rvm.io | bash -s stable
```

The previous command may instruct you to install the required key. Follow
the steps listed and re-run the command. RVM will install and will install
a new bash profile file for you. Either start a new bash session or source
the new profile.

Next we must install the pre-requisisits for a working version of Ruby on
RVM. You should run this as a user with yum privileges:

```bash
rvm requirements
```

Check the latest MRI ruby version available:

```bash
rvm list known
```

At the time of writing the latest was Ruby 2.3. Install that version or later:

```bash
rvm install 2.3
```

Then switch to that version of Ruby and set it as your default (omit
`--default` if that is undesirable):

```bash
rvm use 2.3.0 --default
```

Installing Jekyll and starting a new site
-----------------------------------------

Install the Jekyll gem:

```bash
gem install jekyll
```

Assuming this site is for GitHub pages, create a new site in a folder with the
proper name:

```bash
jekyll new dsclose.github.io
```

Setup a basic Jekyll Git repo:

```bash
cd dsclose.github.io
git init
touch README.md
echo '/_site/' >> .gitignore
echo '.jekyll-metadata' >> .gitignore
git add .
git commit -m 'Initial commit'
```

Push the repo to your GitHub profile.



