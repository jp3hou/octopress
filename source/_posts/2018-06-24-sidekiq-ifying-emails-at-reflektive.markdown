---
layout: post
title: "Sidekiq-ifying Emails at Reflektive"
date: 2018-06-24 11:01:11 -0700
comments: true
categories: [Reflektive, Engineering Practices]
keywords: "jpchou, Julia Chou, Sidekiq, Ruby, Ruby on Rails, Rails, Sidekiq, Newrelic"
description: "Optimizing email delivery and monitoring with Sidekiq and Newrelic"
---

At Reflektive, we offer the option of sending reminder emails after kicking off performance review cycles in order to notify participants of that cycle that they should fill out their reviews. We used <a href='https://github.com/collectiveidea/delayed_job' target='blank'>Delayed::Job</a>, an asynchronous priority queue system pioneered by the Shopify engineering team, to handle the delivery of those emails.

However, we noticed that there was a huge bottleneck when it came to delivering emails to cycles with large numbers of participants. For instance, one company that had approximately 30,000 employees would clog up our worker queues for hours, preventing other companies from sending out their own emails while this large job was still being processed.

One solution might have been to just throw more hardware at it, to spin up more DJ workers to hammer through the job queues. However, we decided to explore the option of using <a href='https://github.com/mperham/sidekiq' target='blank'>Sidekiq</a> as an alternative asynchronous job processing system.

Some of the factors leading to choosing Sidekiq included the fact that Sidekiq has higher concurrency than Delayed::Job because it leverages threads as opposed to single processes.

In addition, Delayed::Job stores its queued jobs in a database table (in our case, Postgres) while they're waiting for a DJ worker to process it. We opted to pair Sidekiq with Redis, an in-memory datastore. This results in a much lower I/O cost because the threads processing Sidekiq jobs don't have to make database queries every time they fetch a new job.

Below are some of Sidekiq's performance statistics (obtained from their Github page)

{% img center /images/sidekiq_performance.png %}

<br/>

<h3>Converting a Delayed::Job to Sidekiq</h3>

The process for converting a Delayed::Job worker to Sidekiq was fairly straightforward. The general structure of our DJ email worker code looked something like this:

```
class DelayedJobEmail
  attr_reader :employee

  def initialize(employee)
    @employee = employee
  end

  def perform
    schedule_email(employee)
  end
end

# Enqueueing this email job:
job = DelayedJobEmail.new(Employee.find(123))
Delayed::Job.enqueue(job)
```

The equivalent Sidekiq implementation of that email worker would look like this:

```
class SidekiqEmail
  include Sidekiq::Worker
  sidekiq_options queue: :process_email

  def perform(employee_id) # Note that Sidekiq does not accept Employee objects
    employee = Employee.find(employee_id)
    schedule_email(employee)
  end
end

# Enqueueing this email job:
SidekiqEmail.perform_async(123)
```

It's worth noting that Sidekiq will perform a JSON dump and load of arguments passed into its workers, so it only accepts pure JSON data types. This increases the consistency of data for edge cases where data has changed from underneath in the time that a job has spent enqueued and waiting for a worker to process it, but it will increase the total number of database calls made.

<br/>

<h3>Testing Sidekiq Jobs</h3>

Of course, for every piece of code, we have to write corresponding tests. Below are various methods of testing Sidekiq jobs in Minitest. They are all different ways of testing the same thing.

```
require 'sidekiq/testing'

class SidekiqEmailTest
  def setup
    Sidekiq::Testing.fake! # Should be fake by default
  end

  def teardown
    Sidekiq::Worker.clear_all # Ensure jobs don't linger between tests
  end

  test '#perform delivers an email (Sidekiq fake mode)' do
    SidekiqEmail.perform_async(123)
    assert_equal(1, SidekiqEmail.jobs.size)
    SidekiqEmailWorker.drain
    # Assert that email was sent properly
  end

  test '#perform delivers an email (Sidekiq inline mode)' do
    Sidekiq::Testing.inline! do
      SidekiqEmail.perform_async(123)
      # Assert that email was sent properly
    end
  end

  test '#perform delivers an email (direct testing)' do
    SidekiqEmail.new.perform(123)
    # Assert that email was sent properly
  end
end
```

<br/>

<h3>Configuring Sidekiq Workers</h3>

Now that we have set up enqueueing Sidekiq jobs, it is a simple matter to spin up a process to start working on those jobs.
The Sidekiq rake command allows you to configure the concurrency, weight, and queues for each process as follows:

```
bundle exec sidekiq -c 20 -q process_email,2 -q deliver_email,1
```

The `-c` flag controls the concurrency of the worker, in this case, 20. The `-q` commands specify which queues this worker will be pulling jobs from, and the `,2` and `,1` after the queue names indicate the weighted priority given to these queues. In this case, jobs from the `process_email` queue will be processed twice as frequently as those from the `deliver_email` queue.

<br/>

<h3>Monitoring Queues</h3>

We use NewRelic to monitor our application's performance, and we wanted to be able to leverage NewRelic to provide further transparency and statistics for our new Sidekiq queues as well. To this end, we set up a Heroku dyno to continually publish our Sidekiq queue sizes as custom events to our NewRelic instance. This way, we can see whether or not a queue is backed up, investigate the causes of any bottlenecks that we observe, and determine whether or not we should allocate more workers to process jobs.

The code we used to publish custom NewRelic events looked something like as follows:

```
module QueueSizeMonitor
  def self.publish_queue_information
    Sidekiq::Queue.all.each do |queue|
      publish(queue.name, queue.size)
    end

    publish('scheduled', ::Sidekiq::ScheduledSet.new.size)
    publish('retries', ::Sidekiq::RetrySet.new.size)
    publish('dead', ::Sidekiq::DeadSet.new.size)
  end

  def self.publish(queue_name, queue_size)
    ::NewRelic::Agent.record_custom_event('QueueSizeMonitor', name: queue_name, size: queue_size)
  end
end
```

And afterwards, we were able to query those custom events in NewRelic and compile a chart to visualize how our Sidekiq queue sizes vary over time.

{% img center /images/sidekiq_queue_size_newrelic.png 600 %}

<br/>

<h3>Results and Next Steps</h3>

After some monitoring, we discovered that switching from Delayed::Job to Sidekiq resulted in about a 4.5x improvement in throughput! We were able to complete a 30,000 employee email campaign in under an hour. We did hit some snags where we discovered that the concurrency we had set for our Sidekiq workers was a little too high and was resulting in a lot of open database connections at once, and we ended up scaling that down.

Moving forward, we are pushing to convert more Delayed::Job work over to Sidekiq.
