---
layout: post
title:  "VoterKarma"
date:   2017-01-25
categories: jekyll update
---

This past weekend, I had the opportunity to participate in the [Debug
Politics](http://www.debugpolitics.com) Hackathon here in NYC.  The event was
put on by a group of designers and developers from San Francisco and hosted at
Casper's office space down by Union Square.  This was the first time that I had
participated in a formal hackathon, and the experience was pretty rewarding,
albeit a little exhausting.

The theme of the hackathon was to address problems or disappointments with the
2016 election cycle.  As you might imagine, this spawned a number of overlapping
ideas about combatting "Fake News", engaging with your representatives, and
increasing the efficacy of activism.  I have spent a fair amount of time
researching and developing thoughts on opportunities within local politics here
in New York, so I led a team with the idea for an application called
**VoterKarma**.

**VoterKarma** is a web application that provides registered New York voters
with an opportunity to see how they would be viewed by modern political
campaigns based upon the information available in the state voter file.  Modern
campaigns use the voter file, and particularly an individual's voter history, to
make decisions about who they will reach out in their targeting and "Get Out the
Vote" efforts. To build the app, we used a combination of the skill sets
available within our team: python sklearn for data analysis, ruby Sinatra for
the web framework, and javascript bootstrap and highcharts for the front-end
design.  In this post, I'll be walking through the Sinatra piece of the
application, although readers are encouraged to visit the
[**VoterKarma**](http://voter-karma.herokuapp.com) to see the full site in
action.

## Why Sinatra?
Before diving into our application, we spent a little time thinking about the
ultimate design requirements up front.  Our biggest constraint was simply time
and lack of resources.  Our four-person team had approximately 48 hours to build
an application from scratch and then present a demo to a panel of judges.  The
actual web application portion of our application - the piece handling requests
and responses and communicating with our voter file database - was actually
pretty simple (for demo purposes at least), so Sinatra fit the bill nicely for
us to get up and running quickly.  

## The Architecture
VoterKarma actually consists of two separate web applications: VoterKarma and
NYCVoterFile.  VoterKarma holds both the user interface and its own database for
storing user data (explained in more detail later).  NYCVoterFile is simply a
web API that allows for access to both the voter file and the voter scores that
we calculated from that data.  We seperated these out for a couple reasons: 1)
we anticipated that access to the voter file data and our associated scores
might have demand beyond just our VoterKarma app; and 2) the information in the
voter file is largely static (occasionally it can be batch-updated after new
elections occur), whereas the VoterKarma is very dynamic and can change based
upon user input.  We felt that it would give us greater flexibility and
scalability to separate out these two pieces, especially if requests on the
NYCVoterFile server grew at a different rate than traffic for VoterKarma.

## VoterKarma
VoterKarma consists of a single controller, a single model and several views.
At its core, the application offers two main features: 1) retrieving an
individual's Voter Score using a query containing a combination of their
first name, last name and date of birth; and 2) allowing a user to "claim" a
record in the voter file by creating a user account linked to that record's New
York State Board of Elections ID (NYSBOEID).

### Retrieving Voter Scores
A user can query the voter file to find an individual's score by completing the
form that is rendered from `get '/score/new'`.  When a new user visits the app,
the root route will redirect them either to this `'/score/new'` page or to the
`'/score'` page based upon whether they are currently logged in.  At the
`'/score/new'` page, the user is prompted to fill in a combination of first
name, last name, and date of birth (YYYYMMDD).  On submission, the form actually
submits a `GET` request to `'/score'`, the rationale being that this request is
actually being forwarded to the NYCVoterFile API and is not actually posting
information to our database.  The controller then either takes the
`current_user` or creates a `User.new` and assigns the request params to that
`User` (namely `firstname, lastname, dob`).  This `User` then calls its
`#get_scores` method, which submits an HTTP request to the VoterFile API and
parses the response to JSON.  Currently, this method passes the results to the
controller and the controller handles the logic for parsing the response
results.  In the future, this business logic should probably be contained within
the `User` model itself, instead of within the controller.  
For example, the controller code currently reads:

```ruby
  get '/score' do
    @user = current_user || User.new
    q_params = params.symbolize_keys.delete_if {|k,v| v.empty?}
    @user.assign_attributes(**q_params)
    resp = @user.get_scores
    if resp[:success]
      @score = Score.new(resp[:body][:voter])
      @averages = Score.new(resp[:body][:averages])
      user_params = resp[:body][:voter].symbolize_keys.delete_if {|k,v| v.empty?}
      @user.firstname = user_params[:firstname]
      @user.lastname = user_params[:lastname]
      @user.dob = user_params[:dob]
      @user.sboeid = user_params[:sboeid]
      erb :'score/index'
    else
      flash[:error] = "Voter Record Not Found"
      erb :'/score/new'
    end
  end
```

Whereas, the logic for handling the score response and updating the `User`
attributes should likely all be handled within the `User#get_scores` method (and maybe
renamed to something like `#get_scores_and_update_attributes`).

```ruby
  get '/score' do
    @user = current_user || User.new
    q_params = params.symbolize_keys.delete_if {|k,v| v.empty?}
    @user.assign_attributes(**q_params)
    if @user.get_scores
      @score = @user.score
      @averages = @user.averages
      erb :'score/index'
    else
      flash[:error] = "Voter Record Not Found"
      erb :'/score/new'
    end
  end
```

Additionally, I could never get the `flash[:error]` to work correctly (not sure
if that was a back-end or front-end issue), but as it stands now the user does
not receive adequate notification that their query has failed (instead just
being sent back to a fresh query form).

### Claiming a Voter Record
From the `'/score'` page, a user can both view the score for their query and
also click a button to claim that record as their own.  If they click the button
to claim their record, they are brought to a `'/signup'` page that shows the
first name, last name, date of birth, and NYSBOEID number as they appear in the
voter file.  The user can then input an email address and password in order to
create a new `User` account that is linked (via the NYSBOEID) to a record in the
voter file.  When returning to the site, the user can then log in using these
credentials and be brought back to their voter record without having to submit
fresh queries.  At this point, there is little other benefit of creating an
account.

### Known Issues
My god, I spent a lot of time trying to figure out the best practice for serving
public assets with Sinatra.  When deploying locally, it is simple enough to just
leave the assets in a public directory, but when deploying on Heroku that wasn't
an option.  Rails and other frameworks use an asset pipeline to compile their
public assets, but I could not figure out how to make that work with Sinatra.
In the end, I ended up just hosting all of my images and javascript on an Amazon
S3 bucket and accessing them through that.  Not ideal and very hard to maintain
(because now my public assets are outside of my version control).

## Conclusion
While we did not win the first or second place recognition at the Hackathon, I
had a great time and would definitely recommend the experience. It was a lot of
fun getting to work with a team of new people and to build something tangible
that we could present at the end of a short cycle.  It can sometimes be
frustrating to have to take shortcuts due to time constraints, but that's a
reality in all walks.
