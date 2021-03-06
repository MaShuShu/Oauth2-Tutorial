**This tutorial was written over 2 years ago, while some of the points may still be valid, I now use [Doorkeeper](https://github.com/applicake/doorkeeper) for my OAuth needs.**

# Oauth2 Demo

Recently I had the need to create an Oauth2 authenticated API.  The following is an app in its most simple form to get you started with creating and testing an Oauth2 powered API, using oauth-plugin, devise and rspec.

## Screencasts

I have created screencasts to go along with this tutorial.  This is my first attempt at screencasting, so please drop me a message if you find them useful or if there is anything you think can be improved.  Your feedback is appreciated.

 * Provider [ogv format](http://screencasts.gazler.com/provider.ogv) [avi format](http://screencasts.gazler.com/provider.avi)
 * Consumer [ogv format](http://screencasts.gazler.com/consumer.ogv) [avi format](http://screencasts.gazler.com/consumer.avi)

## Step 1 - the rails app

Start by opening up your terminal.  For demonstration purposes I recommend creating a folder called oauth to put both the provider and consumer.

    mkdir oauth && cd oauth
    rails new provider
    cd provider
    
The next step is to add the oauth-plugin gem to your Gemfile.  For this demo I will also be using devise for authentication.  If you wish to use RSpec as your testing framework, now would be the time to add it.

    gem 'devise'
    gem "oauth-plugin", ">= 0.4.0.rc2"
    group :test do
        gem 'rspec-rails'
    end
    
You should run *bundle install* to install the oauth-plugin (and rspec.)

If you are using rspec then run:

    rails g rspec:install

If you are using devise then you should create your devise install and user.

    rails generate devise:install
    rails generate devise User
    
Then create the oauth provider (Note I am using rspec)

    rails g oauth_provider --test-framework=rspec
    
And migrate the database
    
    rake db:migrate
    
Might as well do the test database here too

    rake db:test:prepare
    
This will generate some files, there are a few changes required for everything to work.  The first is to delete the file *spec/controllers/oauth_clients_controller_spec.rb* as mentioned in [this commit](https://github.com/pelle/oauth-plugin/commit/6e24ec0ee2f3dc871756b2e8a75fa2181ff504f4).  You should also remove */spec/models/oauth_token_spec.rb* as we are dealing exclusively with oauth2.
The second change is in your *config/routes.rb* file, add:

    root :to => "oauth_clients#index"
    
You will also need to add the following methods to your *app/controllers/application_controller.rb* to make things work as the oauth-plugin gem required a current_user= method.

    def current_user=(user)
      current_user = user
    end

You need to add the following to your user model:

    has_many :client_applications
    has_many :tokens, :class_name=>"Oauth2Token",:order=>"authorized_at desc",:include=>[:client_application]
    
You need to add the following attr_accessor to *app/models/oauth_token.rb*

    attr_accessor :expires_at

The following alias to *app/controllers/oauth_controller.rb* **and** *app/controllers/oauth_clients_controller.rb*

    alias :login_required :authenticate_user!
    
And finally add the following to *config/application.rb*

    require 'oauth/rack/oauth_filter'
    config.middleware.use OAuth::Rack::OAuthFilter
    
For the purposes of this test, we will use fixtures, I recommend using factories for real testing.  Grab the 4 fixtures files out of *spec/fixtures* (I got them from the oauth-plugin but they were not included in the generator)

After these files are included, you can run rspec to test what we have so far.

    bundle exec rspec spec
    
There should be 23 examples, all passing.

You should now create a basic rspec test for what will be your API call.  Grab my one out of *spec/api/v1/data_controller_spec.rb*  Also copy the file *support/api_helper.rb*

When you run rspec on this, it should error, you now need to create your API controller.  Since in this example all the API calls will require a valid oauth token, let's create a base controller and then our data controller.

    rails g controller API::V1::Base
    rails g controller API::V1::Data
    
Change the DataController so it extends API::V1::BaseController

    class Api::V1::DataController < Api::V1::BaseController
    end
    
Now create the routes, add the following to your *config/routes.rb* file
    
    namespace :api do
      namespace :v1 do
        match "data" => "data#show"
      end
    end
    
You will need a show action in your data controller (*app/controllers/api/v1/data_controller*)

    def show
      respond_with ({:super_secret => "oauth_data"})
    end
    
You will also need to specify the formats that your controllers responds to in your base controller (*app/controllers/api/v1/base_controller*)

    respond_to :json, :xml
    
You should also specify which methods require oauth, since it is all in this case, also add the following to your base controllers (the interactive flag is the equivalant of oauth_or_login_required, we want oauth only so we disable it. 

      oauthenticate :interactive=>false

If we run our test specs again now, they should pass and there you have it, the beginnings of an Oauth2 API.

## Step 2 - The client

Change the following in *views/oauth/oauth2_authorize.html.erb*

    <p>Would you like to authorize <%= link_to @token.client_application.name,@token.client_application.url %> (<%= link_to @token.client_application.url,@token.client_application.url %>) to access your account?</p>

To

    <p>Would you like to authorize <%= link_to @client_application.name,@client_application.url %> (<%= link_to @client_application.url,@client_application.url %>) to access your account?</p>

You should now start a rails server and navigate to http://localhost:3000/users/sign_up, after signing up go to http://localhost:3000/oauth_clients and create a client.  Please not that your client callback_url must match that of the one passed through in your app.  If you are using the demo sinatra app, it should be **http://localhost:4567/auth/test**

There are a couple things you should change in *views/oauth_clients/index.html.erb* Change the @tokens block to:

     <% @tokens.each do |token|%>
      <tr>
        <td><%= link_to token.client_application.name, token.client_application.url %></td>
        <td><%= token.authorized_at %></td>
        <td>&nbsp;</td>
      </tr>
    <% end %>

And change the @client_applications block to:

    <% @client_applications.each do |client|%>
      <div>
        <%= link_to client.name, oauth_client_path(client) %>-
          <%= link_to 'Edit', edit_oauth_client_path(client) %>
          <%= link_to 'Delete', oauth_client_path(client), :confirm => "Are you sure?", :method => :delete %>
      </div>
    <% end %>

You should now create a consumer directory outside of the rails root.

    cd ..
    mkdir consumer && cd consumer
    
You will then need to install sinatra and the oauth2 gem **Please note this requires a version of oauth higher than 0.5 **
    gem install sinatra
    gem install oauth2 

Copy the following code, replacing the API keys from those of the client:

    require 'sinatra'
    require 'oauth2'
    require 'json'
    enable :sessions

    def client
      OAuth2::Client.new(consumer_key, consumer_secret, :site => "http://localhost:3000")
    end

    get "/auth/test" do
      redirect client.auth_code.authorize_url(:redirect_uri => redirect_uri)
    end

    get '/auth/test/callback' do
      access_token = client.auth_code.get_token(params[:code], :redirect_uri => redirect_uri)
      session[:access_token] = access_token.token
      @message = "Successfully authenticated with the server"
      erb :success
    end

    get '/yet_another' do
      @message = get_response('data.json')
      erb :success
    end
    get '/another_page' do
      @message = get_response('data.json')
      erb :another
    end

    def get_response(url)
      access_token = OAuth2::AccessToken.new(client, session[:access_token])
      JSON.parse(access_token.get("/api/v1/#{url}").body)
    end


    def redirect_uri
      uri = URI.parse(request.url)
      uri.path = '/auth/test/callback'
      uri.query = nil
      uri.to_s
    end
    
You can grab the required views from *consumer/views*


