# Rspec Best Practices
A collection of Rspec testing best practices

## Table of Contents

* [Describe your methods](#describe-your-methods)
* [Use Context](#use-context)
* [Only one expectation per example](#only-one-expectation-per-example)
* [Test valid, edge and invalid cases](#test-valid-edge-and-invalid-cases)
* [Use let](#use-let)
* [DRY](#dry)
* [Optimize database queries](#optimize-database-queries)
* [Use factories](#use-factories)
* [Choose matchers based on readability](#choose-matchers-based-on-readability)
* [Run specific tests](#run-specific-tests)
* [Debug Capybara tests with save_and_open_page](#debug-capybara-tests-with-save_and_open_page)
* [Only enable JS in Capybara when necessary](#only-enable-js-in-capybara-when-necessary)
* [Consult the logs](#consult-the-logs)
* [Other tips](#other-tips)
* [More Resources](#more-resources)
* [Libraries](#libraries)

## Describe your methods

  Keep clear the methods you are describing using "." as prefix for class methods and "#" as prefix for instance methods.

```ruby
# wrong
describe "the authenticate method for User" do
describe "the save method for User" do

# correct
describe ".authenticate" do
describe "#save" do
```

## Use context

  Use context to organize and DRY up code, keep spec descriptions short, and improve test readability.

```ruby
# wrong
describe User do
  it "should save when name is not empty" do
    User.new(:name => 'Alex').save.should == true
  end

  it "should not save when name is empty" do
    User.new.save.should == false
  end

  it "should not be valid when name is empty" do
    User.new.should_not be_valid
  end

  it "should be valid when name is not empty" do
    User.new(:name => 'Alex').should be_valid
  end
end

# correct
describe User do
  let (:user) { User.new }

  context "when name is empty" do
    it "should not be valid" do
      expect(user.valid?).to be_false
    end

    it "should not save" do
      expect(user.save).to be_false
    end
  end

  context "when name is not empty" do
    let (:user) { User.new(:name => "Alex") }

    it "should be valid" do
      expect(user.valid?).to be_true
    end

    it "should save" do
      expect(user.save).to be_true
    end
  end
end
```

## Only one expectation per example

  Each test example should make only one assertion. This helps you on find errors faster and makes your code easier to read and maintain.

```ruby
# wrong
describe "#fill_gass" do
  it "should have valid arguments" do
    expect { car.fill_gas }.to raise_error(ArgumentError)
    expect { car.fill_gas("foo") }.to_raise_error(TypeError)
  end
end

# correct
describe "#fill_gass" do
  it "should require one argument" do
    expect { car.fill_gas }.to raise_error(ArgumentError)
  end

  it "should require a numeric argument" do
    expect { car.fill_gas("foo") }.to_raise_error(TypeError)
  end
end
```

## Test valid, edge and invalid cases

This is called Boundary value analysis, it’s simple and it will help you to cover the most important cases. Just split-up method’s input or object’s attributes into valid and invalid partitions and test both of them and there boundaries. A method specification might look like that:

```ruby
describe "#month_in_english(month_id)" do
  context "when valid" do
    it "should return 'January' for 1" # lower boundary
    it "should return 'March' for 3"
    it "should return 'December' for 12" # upper boundary
  context "when invalid" do
    it "should return nil for 0"
    it "should return nil for 13"
  end
end
```

## Use let

  When you have to assign a variable to test, instead of using a before each block, [use let](http://stackoverflow.com/questions/5359558/when-to-use-rspec-let). It is memoized when used multiple times in one example, but not across examples.

```ruby
# wrong
describe User, '#locate'
  before(:each) { @user = User.locate }

  it 'should return nil when not found' do
    @user.should be_nil
  end
end

# correct
describe User
  let(:user) { User.locate }

  it 'should have a name' do
    user.name.should_not be_nil
  end
end
```

## DRY
Be sure to apply good code refactoring principles to your tests.

Use before and after hooks:
```ruby
describe Thing do
  before(:each) do
    @thing = Thing.new
  end

  describe "initialized in before(:each)" do
    it "has 0 widgets" do
      @thing.should have(0).widgets
    end

    it "does not share state across examples" do
      @thing.should have(0).widgets
    end
  end
end
```

Extract reusable code into helper methods:
```ruby
# spec/fetures/user_signs_in_spec.rb
require 'spec_helper'

feature 'User can sign in' do
  scenario 'as a user' do
    sign_in

    expect(page).to have_content "Your account"
  end
end

# spec/fetures/user_signs_out_spec.rb
require 'spec_helper'

feature 'User can sign out' do
  scenario 'as a user' do
    sign_in

    click_link "Logout"

    expect(page).to have_content "Sign up"
  end
end
```

```ruby
# spec/support/authentication_helper.rb
module AuthenticationHelper
  def sign_in
    visit root_path

    user = FactoryGirl.create(:user)

    fill_in 'user_session_email',    with: user.email
    fill_in 'user_session_password', with: user.password
    click_button "Sign in"

    return user
  end
end
```

```ruby
# spec/spec_helper.rb
RSpec.configure do |config|
  config.include AuthenticationHelper, type: :feature
  # ...
```

## Optimize database queries

  Large test suites can take a long time to run. Don't load or create more data than necessary.

```ruby
describe User do
  it 'should return top users in User.top method' do
    @users = (1..3).collect { Factory(:user) }
    top_users = User.top(2).all
    top_users.should have(2).entries
  end
end

class User < ActiveRecord::Base
  named_scope :top, lambda { |*args| { :limit => (args.size > 0 ? args[0] : 10) } }
end
```

## Use factories

  Use factory_girl to reduce the verbosity when working with models.

```ruby
# before
user = User.create( :name => "Genoveffa",
                    :surname => "Piccolina",
                    :city => "Billyville",
                    :birth => "17 Agoust 1982",
                    :active => true)

# after
user = Factory.create(:user)
```

## Choose matchers based on readability

RSpec comes with a lot of useful matchers to help your specs read more like language.  When you feel there is a cleaner way … there usually is!

Here are some examples, before and after they are applied:

```ruby
# before: double negative
object.should_not be_nil
# after: without the double negative
object.should be

# before: 'lambda' is too low level
lambda { model.save! }.should raise_error(ActiveRecord::RecordNotFound)
# after: for a more natural expectation replace 'lambda' and 'should' with 'expect' and 'to'
expect { model.save! }.to raise_error(ActiveRecord::RecordNotFound)

# before: straight comparison
collection.size.should == 4
# after: a higher level size expectation
collection.should have(4).items
```

## Run specific tests

Running your entire test suite over and over again is a waste of time.

Run example or block at specified line:
```sh
# in rails
rake spec SPEC=spec/models/demand_spec.rb:30

# not in rails
rspec spec/models/demand_spec.rb:30
```

Run examples that match a given string:
```sh
# in rails
rake spec SPEC=spec/controllers/sessions_controller_spec.rb \
          SPEC_OPTS="-e \"should log in with cookie\""

# not in rails
rspec spec/login_spec.rb -e "should log in with cookie"
```

In Rails, run only your integration tests:
```sh
rake spec:features
```

## Debug Capybara tests with save_and_open_page

Capybara has a `save_and_open_page` method. As the name implies, it saves the page — complete with styling and images — and opens it in your browser so you can inspect it:
```ruby
it 'should register successfully' do
  visit registration_page
  save_and_open_page
  fill_in 'username', :with => 'abinoda'
end
```

## Only enable JS in Capybara when necessary

Only enable JS when your tests require it. Enabling JS slows down your test suite.

```ruby
# only use js => true when your tests depend on it
it 'should register successfully', :js => true do
  visit registration_page
  fill_in 'username', :with => 'abinoda'
end
```
Unless the pages you are testing require JS, it's best to disable JS after you're done writing the test so that the test suite runs faster.

## Consult the logs
When you run any rails application (the webserver, tests or rake tasks), ouput is saved to a log file. There is a log file for each environment: log/development.log, log/test.log, etc.

Take a moment to open up one of these log files in your editor and take a look at its contents. To watch your test log files, use the *nix tool tail:

```sh
tail -f log/test.log
```

Curious what -f means? Check the man page for the tail utility: man tail


## Other tips
* When something in your application goes wrong, write a test that reproduces the error and then correct it. You will gain several hour of sleep and more serenity.
* Use solutions like [guard](https://github.com/guard/guard) (using [guard-rspec](https://github.com/guard/guard-rspec)) to automatically run all of your test, without thinking about it. Combining it with growl, it will become one of your best friends. Examples of other solutions are [test_notifier](https://github.com/fnando/test_notifier), [watchr](https://github.com/mynyml/watchr) and [autotest](http://ph7spot.com/musings/getting-started-with-autotest).
* Use [TimeCop](https://github.com/jtrupiano/timecop) to mock and test methods that relies on time.
* Use [Webmock](https://github.com/bblimke/webmock) to mock HTTP calls to remote service that could not be available all the time and that you want to personalize.
* Use a good looking formatter to check if your test passed or failed. I use [fuubar](http://jeffkreeftmeijer.com/2010/fuubar-the-instafailing-rspec-progress-bar-formatter/), which to me looks perfect.

## More Resources
* [Why Our Code Smells](https://dl.dropboxusercontent.com/u/598519/Why%20Our%20Code%20Smells.pdf)
* [Request Specs and Capybara](http://railscasts.com/episodes/257-request-specs-and-capybara)
* [How I Test](http://railscasts.com/episodes/275-how-i-test)
* [Dmytro best practices](http://kpumuk.info/ruby-on-rails/my-top-7-rspec-best-practices/)
* [Great Rails/Rspec example code](https://github.com/awesomefoundation/awesomebits/tree/master/spec)

## Libraries
* [RSpec](https://github.com/rspec/rspec)
* [Factory girl](https://github.com/thoughtbot/factory_girl)
* [Shoulda Matchers](https://github.com/thoughtbot/shoulda-matchers)
* [Capybara](https://github.com/jnicklas/capybara)
* [Database Cleaner](https://github.com/bmabey/database_cleaner)
* [Spork](https://github.com/sporkrb/spork)
* [Timecop](https://github.com/jtrupiano/timecop)
* [Guard](https://github.com/guard/guard-rspec)
* [Fuubar](https://github.com/jeffkreeftmeijer/fuubar)
* [Webmock](https://github.com/bblimke/webmock)
