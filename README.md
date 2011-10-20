# Abstract

The goal of this guide is to present a set of best practices and style
prescriptions for Ruby on Rails 3 development. It's a complimentary
guide to the already existing community-driven
[Ruby coding style guide](https://github.com/bbatsov/ruby-style-guide).

While in the guide the section **Testing Rails applications** is
after **Developing Rails applications** I'm a firm believer that BDD
is the best way to develop software. Keep that in mind.

Rails is an opinionated framework and this is an opinionated guide. In
my mind I'm totally certain that RSpec is superior to Test::Unit, Sass
is superior to CSS and Haml (Slim) is superior to Erb. So don't expect
to find any Test::Unit, CSS or Erb advice in here.

Some of the advice here is applicable only to Rails 3.1.

**This is guide is in a pre-alpha state! Quite A LOT is missing at
   this point.**

# Developing Rails applications

## Configuration

* Put custom initialization code in `config/initializers`. The code in initializers executes on application startup.
* The initialization code for each gem should be in a separate file with the same name as the gem, for example `carrierwave.rb`, `rails_admin.rb`, etc.


## Routing

* When you need to add more actions to a RESTful resource use
  `member` and `collection` routes.

    ```Ruby
    # bad
    get 'subscriptions/unsubscribe'
    resources 'subscription'

    # good
    resources :subscriptions do
      get 'unsubscribe', :on => :member
    end

    # bad
    get 'photos/search'
    resources :photos

    # good
    resources :photos do
      get 'search', :on => :collection
    end
    ```

* If you need to define multiple `member\collection` routes use the
  alternative block syntax.

    ```Ruby
    resources :subscriptions do
      member do
        get 'unsubscribe'
        # more routes
      end
    end

    resources :photos do
      collection do
        get 'search'
        # more routes
      end
    end
    ```

* Use nested routes to express better the relationship between
  ActiveRecord models.

    ```Ruby
    class Post < ActiveRecord::Base
      has_many :comments
    end

    class Comments < ActiveRecord::Base
      belongs_to :post
    end

    # routes.rb
    resources :posts do
      resources :comments
    end
    ```

* Use namespaced routes to group related actions.

    ```Ruby
    namespace :admin do
      # Directs /admin/products/* to Admin::ProductsController
      # (app/controllers/admin/products_controller.rb)
      resources :products
    end
    ```

* Never use the legacy wild controller route. This route will make all
  actions in every controller accessible via GET requests.

    ```Ruby
    # very bad
    match ':controller(/:action(/:id(.:format)))'
    ```

## Controllers

* Keep the controllers skinny - they should only retrieve data for the
  view layer and shouldn't contain any business logic (all the
  business logic should naturally reside in the model).

## Models

* Introduce non-ActiveRecord model classes freely.

### ActiveRecord

* Avoid altering ActiveRecord defaults (table names, primary key, etc)
  unless you have a very good reason (like a database that's not under
  your control).
* Group macro-style methods (`has_many`, `validates`, etc) in the
  beginning of the class definition.
* Always use the new
  ["sexy" validations](http://thelucid.com/2010/01/08/sexy-validation-in-edge-rails-rails-3/).
When a custom validation is used more than once or the validation is some regular expression mapping, 
create a custom validator file.

    ```Ruby
    # bad
    class Person
      validates :email, format: { with: /^([^@\s]+)@((?:[-a-z0-9]+\.)+[a-z]{2,})$/i } 
    end

    # good
    class EmailValidator < ActiveModel::EachValidator
      def validate_each(record, attribute, value)
        record.errors[attribute] << (options[:message] || "is not a valid email") unless value =~ /^([^@\s]+)@((?:[-a-z0-9]+\.)+[a-z]{2,})$/i
      end
    end


    class Person
      validates :email, email: true
    end
    ```

* Use named scopes freely.
* When a named scope with lambda and parameters becomes too complicated it is better to make a class method instead which 
serves the same purpose of the named scope and returns and `ActiveRecord::Relation` object.


## Migrations

* Keep the schema.rb under version control.
* Avoid setting defaults in tables themselves. Use the model layer
  instead.

    ```Ruby
    def amount
      read_attributed(:amount) or 0
    end
    ```

* When writing constructive migrations (adding tables or columns), use the new Rails 3.1 way of doing the migrations - use the `change` method instead of `up` and `down` methods.

## Views

* Never call the model layer directly from a view.
* Never make complex formatting in the views, export the formatting to a method in the view helper or the model.
* Mitigate code duplication by using partial templates and layouts.
* Add client side validation for the custom validators. The steps to do this are:
  * Declare a custom validator which extends `ClientSideValidations::Middleware::Base`

    ```Ruby
    module ClientSideValidations::Middleware
      class Email < Base
        def response
          if request.params[:email] =~ /^([^@\s]+)@((?:[-a-z0-9]+\.)+[a-z]{2,})$/i
            self.status = 200
          else
            self.status = 404
          end
          super
        end
      end
    end
    ```

  * Create a new file `public/javascripts/rails.validations.custom.js.coffee` and add a reference to it in your `application.js.coffee` file:

    ```Ruby
    # app/assets/javascripts/application.js.coffee
    #= require rails.validations.custom
    ```

  * Add your client-side validator:

    ```Ruby
    #public/javascripts/rails.validations.custom.js.coffee
    clientSideValidations.validators.remote['email'] = (element, options) ->
      if $.ajax({
        url: '/validators/email.json',
        data: { email: element.val() },
        async: false
      }).status == 404
        return options.message || 'invalid e-mail format'
    ```


## Mailers

* Name the mailers `SomethingMailer`.
* Provide both HTML and plain-text view templates.
* Enable errors raised on failed mail delivery in your development environment. The errors are disabled by default.

    ```Ruby
    # config/environments/development.rb

    config.action_mailer.raise_delivery_errors = true
    ```

* Use `smtp.gmail.com` for SMTP server in the development environment.


    ```Ruby
    # config/environments/development.rb

    config.action_mailer.smtp_settings = {
      :address => "smtp.gmail.com",
      # more settings
    }  
    ```

* Provide default settings for host name.

    ```Ruby
    # config/environments/development.rb
    config.action_mailer.default_url_options = {:host => "#{local_ip}:3000"}


    # config/environments/production.rb
    config.action_mailer.default_url_options = {:host => "your_site.com"}

    # in your mailer class
    default_url_options[:host] = 'your_site.com'
    ```

* If you need to use a link to your site in an email, always use the `_url`, not `_path` methods. The `_url` methods include the host name and the `_path` don't.

    ```Ruby
    # wrong
    You can always find more info about this course 
    = link_to "here", url_for(course_path(@course))

    # right
    You can always find more info about this course
    = link_to "here", url_for(course_url(@course))
    ```

* Format the from and to addresses properly. Use the following format: 

    ```Ruby
    # in your mailer class
    default from: 'Your Name <info@your_site.com>'
    ```

## Bundler

* Put gems used only for development or testing in the appropriate group in the Gemfile.
* Avoid OS specific gems. Bundler has no facilities to deal with those properly.
* Use only established gems in your projects. If you're contemplating
on including some little-known gem you should do a careful review of
its source code first.
* Do not remove the `Gemfile.lock` from version control. This is not
  some randomly generated file - it makes sure that all of your team
  members get the same gem versions when they go a `bundle install`.

## Managing processes

* Use foreman.

# Testing Rails applications

The best approach to implementing new features is probably the BDD
approach. You start out by writing some high level feature tests (
generally written using Cucumber ), then you use this tests to drive
out the implementation of the feature. First you write view specs for
the feature and use those specs to create the relevant
views. Afterwards you create the specs for the controller(s) that will
be feeding data to the views and use those specs to implement the
controller. Finally you implement the models specs and the models
themselves.

## Cucumber

* Tag your pending scenarios with `@wip` (work in progress).  These scenarios will not be taken into account and will not be marked as failing.
When finishing the work on a pending scenario and implementing the functionality it tests, 
the tag `@wip` should be removed in order to include this scenario in the test suite.
* Setup your default profile to exclude the scenarios tagged with `@javascript`. 
They are testing using the browser and diabling them is recommended to increase the regular test execution.  
* Setup a separate profile for the scenarios marked with `@javascript` tag.
  * The profiles can be configured in the `cucumber.yml` file. 

    ```Ruby
    # definition of a profile:
    profile_name: --tags @tag_name features
    ```

  * A profile is run with the command:

    ```
    cucumber -p profile_name        
    ```

* If using [fabrication](http://fabricationgem.org/) for fixtures replacement, use the predefined [fabrication steps](http://fabricationgem.org/#!cucumber-steps)
* Before writing your own steps for elements selection, first check the `websteps.rb` file under the `step_definitions` directory and reuse some of the existing steps if possible.
* When checking for the presence of an element with visible text (link, button, etc.) check for the text, not the element id. This can detect problems with the i18n.
* Create separate features for different functionality regarding the same kind of objects:

    ```Ruby
    # bad
    Feature: Articles
    # ... feature  implementation ...

    # good
    Feature: Article Editing
    # ... feature  implementation ...

    Feature: Article Publishing
    # ... feature  implementation ...

    Feature: Article Search
    # ... feature  implementation ...

    ```
* Each feature has three main components
  * Title 
  * Narrative - a short explanation what the feature is about. 
  * Acceptance criteria - the set of scenarios each made up of individual steps.
* The most common format is known as the Connextra format. 

    ```Ruby
    In order to [benefit] ...
    A [stakeholder]...
    Wants to [feature] ...
    ```

This format is the most common but is not required, the narrative can be free text depending on the complexity of the feature.

* Use Scenario Outlines freely to keep the scenarios DRY.

    ```Ruby
    Scenario Outline: Entering invalid e-mail displays an error
      Given I am on the registration page
      And I fill in "E-mail" with "<email>"
      And I should press "Register"
      Then I should see "<error>"
    
    Examples: 
      |email         |error                 |
      |              |The e-mail is required|
      |invalid email |is not a valid e-mail |
    ```

* The steps for the scenarios are in `.rb` files under the `step_definitions` directory. The naming convention for the steps file is `[description]_steps.rb`.
The steps can be separated into different files based on different criterias. It is possible to have one steps file for each feature (`home_page_steps.rb`). 
There also can be one steps file for all features for a particular object (`articles_steps.rb`).
* Use multiline step arguments to avoid repetition

    ```Ruby
    Scenario: User profile
      Given I am logged in as a user with e-mail "user@test.com"
      When I go to the profile edit page
      Then I should see the following information:
        |First name|John         |
        |Last name |Doe          |
        |E-mail    |user@test.com|


    # the step:
    Then "I should see the following information:" do |table|
      table.raw.each do |field, value|
        field = find_field(field)
        field.value.should =~ /#{value}/
      end
    end    
    ```

* Use compound steps to keep the scenario DRY

    ```Ruby
    # ...
    When I register as a user with email "user@test.com" and password "secret"
    # ...

    # the step:
    When /^I register as a user with email "([^"]*)" and password "([^"]*)"$/ do |email, password|
      steps %Q{
        When I go to the user registration page
        And I fill in "E-mail" with #{email}
        And I fill in "Password" with #{password}
        And I fill in "Password Confirmation" with #{password}
        And I click "Register"
      }
    end    
    ```


## RSpec

* Use just one expectation per example.

    ```Ruby
    # bad
    describe ArticlesController do
      #...

      describe "GET new" do
        it "assigns new article and renders the new article template" do
          get :new
          assigns[:article].should be_a_new Article
          response.should render_template :new
        end
      end

      # ...
    end

    # good
    describe ArticlesController do
      #...

      describe "GET new" do
        it "assigns a new article" do
          get :new
          assigns[:article].should be_a_new Article
        end

        it "renders the new article template" do
          get :new
          response.should render_template :new
        end
      end

      # ...
    end
    ```

* Make heavy use of `describe` and `context`
* Name the `describe` blocks as follows:
  * use “description” for non-methods
  * use pound “#method” for instance methods
  * use dot “.method” for class methods

    ```Ruby
    class Article
      def summary
        #...
      end

      def self.latest
        #...
      end
    end

    # the spec...
    describe Article 
      describe "#summary"
        #...
      end

      describe ".latest"
        #...
      end      
    end
    ```

* Use [fabricators](http://fabricationgem.org/) to create test objects
* Make heavy use of mocks and stubs

    ```Ruby
    # mocking a model
    article = mock_model(Article)

    # stubbing a method
    Article.stub(:find).with(article.id).and_return(article)    
    ```
  
* Use `let` blocks instead of `before(:all)` blocks to create data for
  the spec examples. `let` blocks get lazily evaluated.

    ```Ruby
    # use this:
    let(:article) { Fabricate(:article) }

    # ... instead of this:
    before(:each) { @article = Fabricate(:article) }
    ```

* Use `subject` when possible

    ```Ruby
    describe Article do
      subject { Fabricate(:article) }

      it "is not published on creation" do
        subject.should_not be_published
      end
    end
    ```

* Use `specify` if possible. It is a sinonym of `it` but is more readable when there is no docstring.

    ```Ruby
    # bad
    describe Article do
      before { @article = Fabricate(:article) }

      it "is not published on creation" do
        @article.should_not be_published
      end
    end

    #good
    describe Article do
      let(:article) { Fabricate(:article) }
      specify { article.should_not be_published }
    end
    ```

* Use `its` when possible

    ```Ruby
    # bad
    describe Article do
      subject { Fabricate(:article) }

      it "has the current date as creation date" do
        subject.creation_date.should == Date.today
      end
    end

    #good
    describe Article do
      subject { Fabricate(:article) 
      its(:creation_date) { should == Date.today }
    end
    ```


### Views

### Controllers

* Mock the models and stub their methods. Testing the controller should not depend on the model creation.
* Test only the behaviour the controller should be responsible about:
  * Execution of particular methods
  * Data returned from the action - assigns, etc.
  * Result from the action - template render, redirect, etc.

    ```Ruby
    # Example of a commonly used controller spec
    # spec/controllers/articles_controller_spec.rb
    # We are interested only in the actions the controller should perform
    # So we are mocking the model creation and stubbing its methods
    # And we concentrate only on the things the controller should do

    describe ArticlesController do      
      # The model will be used in the specs for all methods of the controller
      let(:article) { mock_model(Article) }

      describe "POST create" do
        before { Article.stub(:new).and_return(article) }

        it "creates a new article with the given attributes" do
          Article.should_receive(:new).with("title" => "The New Article Title").and_return(article)
          post :create, :message => { "title" => "The New Article Title" }
        end

        it "saves the article" do
          article.should_receive(:save)
          post :create
        end
 
        it "redirects to the Articles index" do
          article.stub(:save)
          post :create
          response.should redirect_to(:action => "index")
        end
      end 
    end
    ```

* Use context when the controller action has different behaviour depending on the received params.

    ```Ruby
    # A classic example for use of contexts in a controller spec is creation or update when the object saves successfully or not.

    describe ArticlesController do      
      let(:article) { mock_model(Article) }

      describe "POST create" do
        before { Article.stub(:new).and_return(article) }

        it "creates a new article with the given attributes" do
          Article.should_receive(:new).with("title" => "The New Article Title").and_return(article)
          post :create, :article => { "title" => "The New Article Title" }
        end

        it "saves the article" do
          article.should_receive(:save)
          post :create
        end

        context "when the article saves successfully" do
          before { article.stub(:save).and_return(true) }

          it "sets a flash[:notice] message" do
	    post :create
            flash[:notice].should eq("The article was saved successfully.")
          end

          it "redirects to the Articles index" do
            post :create
            response.should redirect_to(:action => "index")
          end
        end

        context "when the article fails to save" do
          before { article.stub(:save).and_return(false) }
    
          it "assigns @article" do
            post :create
            assigns[:article].should be_eql(article)
          end
    
          it "re-renders the 'new' template" do
            post :create
            response.should render_template("new")
          end
        end
      end 
    end
    ```

### Models

### Mailers

### Uploaders

# Contributing

Nothing written in this guide is set in stone. It's my desire to work
together with everyone interested in Rails coding style, so that we could
ultimately create a resource that will be beneficial to the entire Ruby
community.

Feel free to open tickets or send pull requests with improvements. Thanks in
advance for your help!

# Spread the Word

A community-driven style guide is of little use to a community that
doesn't know about its existence. Tweet about the guide, share it with
your friends and colleagues. Every comment, suggestion or opinion we
get makes the guide just a little bit better. And we want to have the
best possible guide, don't we?
