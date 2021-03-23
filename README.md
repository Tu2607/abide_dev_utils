# AbideDevUtils

Helper library and CLI app for Abide development.

## Features

### CLI

* Type `abide -h` to see the CLI help menu
* All commands have help menus
* Tested on MacOS, should also work on Linux

### PDK-like manifest generation from local ERB templates

* Create a directory in your module called `object_templates` to hold ERB files
* Generate manifests from those ERB files

### XCCDF to Hiera conversion

* Convert the contents of an XCCDF XML file to Hiera

### Coverage Reporting

* Generate a coverage report (i.e. how many CIS controls are covered by classes in your module)

### Jira Integration

* Create Jira issues in bulk from coverage reports

### Supports Configuration via Local YAML file

* Fully configurable via the `~/.abide_dev.yaml` file

## Installation

### CLI app

Install from RubyGems:

`gem install abide_dev_utils`

Then to access the help menu:

`abide -h`

### As a dependency

Add this line to your application's Gemfile:

```ruby
gem 'abide_dev_utils'
```

And then execute:

`$ bundle install`

Or install it yourself as:

`$ gem install abide_dev_utils`

### Build it yourself

Clone this repo:

`git clone <this repo>`

Build the gem:

`bundle exec rake build`

The gem will be located at `pkg/<gem file>`

Install the gem:

`gem install pkg/<gem file>`

## Usage

* `abide -h` - CLI top-level help menu
* `abide <command> -h` - Command-specific help menu

### Overview of Commands

* `abide jira` - Command namespace for Jira commands
  * `abide jira auth` - Authenticate with Jira. Only useful as a stand-alone command to test authentication
  * `abide jira from_coverage` - Creates a parent issue with subtasks from a Puppet coverage report
  * `abide jira get_issue` - Gets a specific Jira issue
  * `abide jira new_issue` - Creates a new Jira issue
* `abide puppet` - Command namespace for Puppet commands
  * `abide puppet coverage` - Generate a "coverage" report. This examines how many manifests you have in your Puppet module versus how many CIS controls exist in the local Hiera data and gives you a breakdown of what percentage of the selected CIS benchmark / profile you have successfully covered.
  * `abide puppet new` - Generate a new manifest from a local ERB template. Functions similar to `pdk new` except you can create arbitrary manifests.
* `abide test` - **BUGGED** Runs a module's test suite. Currently has issues with local gem environments.
* `abide xccdf` - Command namespace for XCCDF commands
  * `abide xccdf to_hiera` - Converts a benchmark XCCDF file to a Hiera yaml file

### Jira Command Reference

#### from_coverage

* Required positional parameters:
  * `REPORT` - A path to a JSON file generated by the `abide puppet coverage` command
  * `PROJECT` - A Jira project code. This is typically an all-caps abbreviation of a Jira project
* Options:
  * `--dry-run`, `-d` - Prints the results of this command to the console instead of actually creating Jira issues

### Puppet Command Reference

#### coverage

* Required positional parameters:
  * `CLASS_DIR` - The path to the directory in your module that holds manifests named after the benchmark controls
  * `HIERA_FILE` - The path to the Hiera file generated by the `abide xccdf to_hiera` command
* Options:
  * `--out-file`, `-o` - The path to save the coverage report JSON file
  * `--profile`, `-p` - A benchmark profile to generate the report for. By default, generates the report for all profiles

#### new

* Required positional parameters:
  * `TYPE` - The type of object you would like to generate. This value is the name of an ERB template without `.erb` located in your template directory
  * `NAME` - The fully namespaced name of the new object. Subdirectories will be automatically created within `manifests` based on the namespacing of this parameter if the directories do not exist.
* Options:
  * `--template-dir`, `-t` - Name of the template directory relative to your module's root directory. Defaults to `object_templates`
  * `--root-dir`, `-r` - Path to the root directory of your module. Defaults to the current working directory
  * `--absolute-template-dir`, `-A` - Allows you to specify an absolute path with `--template-dir`. This is useful if your template directory is not relative to your module's root directory
  * `--template-name`, `-n` - Allows you to specify a template name if it is different than the `TYPE` parameter
  * `--vars`, `-V` - Comma-separated key=value pairs to pass in to the template renderer. This allows you to pass arbitrary values that can be used in your templates.

`abide puppet new` exposes a few variables for you to use in your templates by default:

* `@obj_name` - The value of the `NAME` parameter passed into the command
* `@vars` - A hash of any key=value pairs you passed into the command using `--vars`

Examples:

Lets assume we have a module at `~/test_module` and we have a directory in that module called `object_templates`. In the template directory, we have two ERB template files: `control_class.erb` and `util_class.erb`.

* control_class.erb

```text
# @api private
class <%= @obj_name %> (
  Boolean $enforce = true,
  Hash $config = {},
) {
  if $enforce {
    warning('Class not implemented yet')
  }
}

```

* util_class.erb

```text
class <%= @obj_name %> (
<% if @vars.key?('testvar1') -%>
  $testparam1 = '<%= @vars['testvar1'] %>',
<% else -%>
  $testparam1 = undef,
<% end -%>
) {
  file { $testparam1:
    ensure => file,
    mode   => '<%= @vars.fetch('testvar2', '0644') %>',
  }
}

```

```text
$ cd ~/test_module
$ ls object_templates
control_class.erb  util_class.erb
$ ls manifests
init.pp
$ abide puppet new control_class 'test_module::controls::test_new_control'
Created file /Users/the.dude/test_module/manifests/controls/test_new_control.pp
$ abide puppet new util_class 'test_module::utils::test_new_util' -v 'testvar1=dude,testvar2=sweet'
Created file /Users/the.dude/test_module/manifests/utils/test_new_util.pp
$ cat manifests/controls/test_new_control.pp
# @api private
class test_module::controls::test_new_control (
  Boolean $enforce = true,
  Hash $config = {},
) {
  if $enforce {
    warning('Class not implemented yet')
  }
}

$ cat manifests/utils/test_new_util.pp
class test_module::utils::test_new_util (
  $testparam1 = 'dude',
) {
  file { 'dude':
    ensure => file,
    mode   => 'sweet',
  }
}

```

### XCCDF Command Reference

#### to_hiera

* Required positional parameters:
  * `XCCDF_FILE` - Path to an XCCDF XML file
* Options:
  * `--benchmark-type`, `-b` - The type of benchmark. Defaults to `cis`
  * `--out-file`, `-o` - A path to a file where you would like to save the generated Hiera
  * `--parent-key-prefix`, `-p` - Allows you to append a prefix to all top-level Hiera keys

## Development

After checking out the repo, run `bin/setup` to install dependencies. Then, run `rake spec` to run the tests. You can also run `bin/console` for an interactive prompt that will allow you to experiment.

To install this gem onto your local machine, run `bundle exec rake install`. To release a new version, update the version number in `version.rb`, and then run `bundle exec rake release`, which will create a git tag for the version, push git commits and the created tag, and push the `.gem` file to [rubygems.org](https://rubygems.org).

## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/hsnodgrass/abide_dev_utils.

## License

The gem is available as open source under the terms of the [MIT License](https://opensource.org/licenses/MIT).
