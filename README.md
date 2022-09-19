# Rails Awesome Tests

Inspired by [wroc_love.rb 2022](https://wrocloverb.com/) presentation, this is a collection of tools and advices how to shorten feedback loop from your tests.

# Presentation

Presentation went how through the 1960s, 1970s and 1980s we shortened our feedback loop by moving from indirect to direct access to our computers and automated testing. However, for some developers expected code change can give feedback loop in similiar times to those like in 1960s or later. This is because we still tend to delegate running our code to shared environments (like CI servers) and making our local setups unfit for exploratory programming and debugging. 

## FEEDBACK LOOP SHORTENING 101

* Get a faster computer
* Run stuff locally
* Get results quicker


## Get faster computer

Measure how often you wait for your hardware during development in testing. Consider multiplying this time by your hourly rate and see what upgrades make sense. Remember that CI is also a computer that you use, so include it in those measurements.

Try to answer the following questions:

 * How many builds are waiting in the queue?
 * How many stale PRs need to be rebased before they can get merged?
 * How much is a faster CI server?

**Reducing 10 minutes of test suite, on span of 5 test runs per day (1000 minutes) can save ~530 USD/monthly from developer cost**

### [GitHub self-hosted runners](https://docs.github.com/en/actions/hosting-your-own-runners/about-self-hosted-runners)

A good cost effective solution to slow CI is getting dedicated server and installing docker and simple custom script to run tests. This dedicated machine can often have 2x or 4x more power than your current CI and still be cheaper.

## Run stuff locally

1. Allow for prototyping:
   * Mock external services early, make sure other developers can use those mocks without any setup
   * Write code and test in rails console, even on production if you need to validate your assumptions about the data
   * Use [feature flags](https://github.com/jnunemaker/flipper) to hide unfinished features.
   
2. Simplify your setup
   * Dockerize the dependencies, even if you don't develop your application in docker
   * Provide defaults for all the ENV variables
   * Reduce the number of setup steps:

   `bundle && db:setup && rails c` is preferable, `docker-compose up` (or equivalent) is also good.

3. Scrutinize your dependencies (Gemfile)
   * How many gems are in your Gemfile? When did you last time check if you can prune some of them?
   * How many configurations and default values are there? Are there any rspec/minitest configs for those?
4. Plant seeds
   * Good seeds help to get application in usable state quicker
   * Having users with *known* passwords make developers use the same accounts with the same settings
   * Having a complete set of seeds allows for easy resetting of the state when jumping between branches and working on different tasks

5. Don't ignore docker
   * Docker images are dependencies too
   * Big docker images are slow to download, [try to make them smaller](https://phoenixnap.com/kb/docker-image-size), make sure they are cached on CI

## Get results quicker
Speed up your tests: **15 minutes full suite tops**. The longer it takes to run the tests the more focus the developers lose.

### `ag sleep spec` 
 * Don’t sleep in your tests, especially in Capybara tests, since [they provide much better solutions](https://www.varvet.com/blog/why-wait_until-was-removed-from-capybara/).
 * If your code is dependant on sleep, you can never optimize it - it will break on slow machines, and it will waste sleep time on fast machines.

### Grep your specs for other things
 * Loops, especially dynamically creating tests
   * they usually test the same thing, but with different data, they are very easy to extend, so developers will add more items over time
 * shared_context & shared_example 
   * again, very easy to add in multiple places, often testing the same underlying implementation
 * File loading
   * use small files if testing, adding 25mb user avatar thumbnail is a good idea to slow down whole suite, especially on slow CI server

### Parallel testing
Often chosen as first lifeline to speed up tests. It is a good idea, but it is not a silver bullet. It can be a good idea to run tests in parallel. 

On downside, unoptimized parallel tests will not shorten feedback loop, but will increase it. Split your tests evenly, if one test is 10x slower than other, it will slow down whole suite.

Other downsides:

 * Setup takes more time (consider setting up database once and cloning images, benchmark what makes sense for your use case)
 * Separation isn’t always complete, look for file system access, databases and cache (ElasticSearch, Redis and/or Memcached)
 * If you have a small subset of very slow tests you might not benefit from it

Some tools to help you with parallel testing:
 * <https://github.com/grosser/parallel_tests> - make sure you use previous run timings to get a more even split
 * <https://knapsackpro.com/>
   * optimizes parallel worker so that each gets about same time for all tests, paid solution
 * <https://github.com/tmm1/test-queue>
   * really good locally, but troublesome to setup on CI
     * if anyone has a good guide, please share

### Lean setup
For tests where some endpoint takes a long time, you want to limit amount of times you execute the slow code, but putting all expectations in one block will not give you all failure messages because it will stop on first one.

**Unless you use [`:aggregate_failures`](https://relishapp.com/rspec/rspec-core/docs/expectation-framework-integration/aggregating-failures) flag** this flag will collect all failures and report them for this one run.

```ruby
let! (:gazillion_records_worth_of_data) { create(:life_universe_and_everything) }
before { get '/some-heavy-endpoint?with=a_complex_filter'}
it 'has a valid response', :aggregate_failures do
  expect (response).to have_http_status (200)
  expect (json_response[:errors]).to be_nil
  expect (json_response[:items]&.first).to eq some_object
end
```

This approach works great for grouping tests with the same setup, but different expectations.

### TestProf by EvilMartians
<https://test-prof.evilmartians.io/#/> is a collection of different tools to analyze your test suite performance. This isn't a specific tool but a guide how to approach slow testing suite.

GitHub: <https://github.com/test-prof/test-prof>

#### Factory cascades
Let’s create a user, which has to be in a company, which needs an address, which needs a country,
countries have currencies, currencies should have some payments, which need some invoices.

```
$ EVENT PROF="factory.create" bundle exec rspec
In the output, you see the total time spent on creating records
from factories and top five slowest specs:
[TEST PROF INFO] EventProf results for factorv.create
Total time: 03:07.353
Total events: 7459
Top 5 slowest suites (by time):
UsersController (users_controller_spec.rb:3) •
- 00:10.119 (581 / 248)
DocumentsController (documents_controller_spec.rb:3) - 00:07.494 (71.
24)
RolesController (roles controller spec.rb:3) - 00:04.972 (181 / 76)
Finished in 6 minutes 36 seconds (files took 32.79 seconds to load)
3158 examples, 0 failures, 7 pending
```

Source: <https://evilmartians.com/chronicles/testprof-2-factory-therapy-for-your-ruby-tests-rspec-minitest>

**Remove unnecessary associations, make them explicitly in #create**
```ruby
factory :comment do
  sequence(:body) { |n| "Awesome comment ##{n}" }
  # do not declare associations
  # author
  # answer
end
```

**Use transient properties and callbacks to set associations**
`TODO`

**Use create_default helper to re-use objects**
`TODO`

#### Remove IO intensive operations on per-test basis
Clearing up tmp/cache on each test will slow down your suite. You can do it after whole suite.

### DB or not DB
In cases where you are not using database (callbacks, saving or quering) just `build` or `build_stubbed` models. It will be much faster.

```ruby
let(:valid_pony) { create(:pony, size: 5) } # change create to build
it 'requires a valid size' do
  expect(valid_pony.size).not_to be_nil
end
```

TODO: I remeber there was a tool to automatically change `create` to `build` in tests, and revert the change for tests that fail.

### Building test objects hierarchy
 * build_stubbed
 * build
 * create # no associations
 * create

### Crystalball
<https://github.com/toptal/crystalball> is regression test selection library for your RSpec test suite. What it does, is collect information, which specific tests cover specific models, controllers. This allows to backtrack which tests need to run if You modify specific files.

### Stubbing Paperclip Convert
If You have styles defined for your images in paperclip, each time you create user, thumbnail will be converted to 3,4 or 5 different files. That is slow, stub it by default, and only check it once.

```ruby
# spec/support/paperclip.rb
RSpec.configure do |config|
  config.before(:each) do |example|
    unless example.metadata[:with].try(:include?, :original_paperclip)
      allow_any_instance_of(Paperclip::Attachment).to receive(:post_process)
      allow(Paperclip).to receive(:run).and_call_original
      allow(Paperclip).to receive(:run).with("convert").and_return(nil)
    end
  end
end
```

### Run the database in RAM
If you are bound by database performance in tests, you can use this lifeline to speed up your tests.

**This uses RAM to store data in memory, so it’s not recommended when files are not cleaned up and might grow bigger than Your machine’s RAM. It will be cleared when container will STOP**

```yml
version: '3'
services:
  mysqldb:
    image: mysql:5.7
    restart: unless-stopped
    environment:
      - MYSQL_ALLOW_EMPTY_PASSWORD=true
    ports:
      - "3306:3306"
    tmpfs:
      - /var/lib/mysql
```

Or if You are compatibile with SQLite, then use this `database.yml`:
```yml
test:
  adapter: sqlite3
  database: ":memory"
```

