# Fluentd Cloudwatch Plugin [![Circle CI](https://circleci.com/gh/sampointer/fluent-plugin-cloudwatch-ingest.svg?style=shield)](https://circleci.com/gh/sampointer/fluent-plugin-cloudwatch-ingest) [![Gem Version](https://badge.fury.io/rb/fluent-plugin-cloudwatch-ingest.svg)](https://badge.fury.io/rb/fluent-plugin-cloudwatch-ingest) ![](http://ruby-gem-downloads-badge.herokuapp.com/fluent-plugin-cloudwatch-ingest?type=total)

## Introduction

This gem was created out of frustration with existing solutions for Cloudwatch log ingestion into a Fluentd pipeline. Specifically, it has been designed to support:

* The 0.14.x fluentd plugin API
* Native IAM including cross-account authentication via STS
* Tidy state serialization
* HA configurations without ingestion duplication

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'fluent-plugin-cloudwatch-ingest'
```

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install fluent-plugin-cloudwatch-ingest

## Usage
```
<source>
  @type cloudwatch_ingest
  region us-east-1
  sts_enabled true
  sts_arn arn:aws:iam::123456789012:role/role_in_another_account
  sts_session_name fluentd-dev
  aws_logging_enabled true
  log_group_name_prefix /aws/lambda
  log_stream_name_prefix 2017
  limit_events 10000
  state_file_name /mnt/nfs/cloudwatch.state
  interval 60
  api_interval 5           # Time to wait between API call failures before retry
  limit_events 10000       # Number of events to fetch in any given iteration
  max_event_age_minutes 0  # Do not fetch events with age older than this amount of minutes
  <parse>
    @type cloudwatch_ingest
    expression /^(?<message>.+)$/
    time_format %Y-%m-%d %H:%M:%S.%L
    event_time true         # take time from the Cloudwatch event, rather than parse it from the body
    inject_group_name true  # inject the group name into the record
    inject_stream_name true # inject the stream name into the record
  </parse>
</source>
```

### Authentication
The plugin will assume an IAM instance role. Without either of the `sts_*` options that role will be used for authentication. With those set the plugin will
attempt to `sts:AssumeRole` the `sts_arn`. This is useful for fetching logs from many accounts where the fluentd infrastructure lives in one single account.

### Prefixes
Both the `log_group_name_prefix` and `log_stream_name_prefix` may be omitted, in which case all groups and streams will be ingested. For performance reasons it is often desirable to set the `log_stream_name_prefix` to be today's date, managed by a configuration management system.

### State file
The state file is a YAML serialization of the current ingestion state. When running in a HA configuration this should be placed on a shared filesystem, such as EFS.
The state file is opened with an exclusive write call and as such also functions as a lock file in HA configurations. See below.

### HA Setup
When the state file is located on a shared filesystem an exclusive write lock will attempted each `interval`.
As such it is safe to run multiple instances of this plugin consuming from the same CloudWatch logging source without fear of duplication, as long as they share a state file.
In a properly configured auto-scaling group this provides for uninterrupted log ingestion in the event of a failure of any single node.

### Sub-second timestamps
When using `event_time true` the `@timestamp` field for the record is taken from the time recorded against the event by Cloudwatch. This is the most common mode to run in as it's an easy path to normalization: all of your Lambdas or other AWS service need not have the same, valid, `time_format` nor a regex that matches every case.

If your output plugin supports sub-second precision (and you're running fluentd 0.14.x) you'll "enjoy" sub-second precision.

#### Elasticsearch
It is a common pattern to use fluentd alongside the [fluentd-plugin-elasticsearch](https://github.com/uken/fluent-plugin-elasticsearch) plugin, either directly or via [fluent-plugin-aws-elasticsearch-service](https://github.com/atomita/fluent-plugin-aws-elasticsearch-service), to ingest logs into Elasticsearch.

At present there is a bug within this plugin that, via an unwise cast, causes records without a named timestamp field to be cast to `DateTime`, losing the precision. This PR: https://github.com/uken/fluent-plugin-elasticsearch/pull/249 fixes that issue. If you need this functionality then I would urge you to comment and express interest over there.

Failing that I maintain my own fork of that repository with the fix in place: https://github.com/sampointer/fluent-plugin-elasticsearch/tree/add_configurable_time_precision_when_timestamp_missing

### IAM
IAM is a tricky and often bespoke subject. Here's a starter that will ingest all of the logs for all of your Lambdas in the account in which the plugin is running:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "logs:DescribeLogGroups",
                "logs:DescribeLogStreams",
                "logs:DescribeMetricFilters",
                "logs:FilterLogEvents",
                "logs:GetLogEvents"
            ],
            "Resource": [
                "arn:aws:logs:eu-west-1:123456789012:log-group:/aws/lambda/*:*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "logs:DescribeLogGroups",
            ],
            "Resource": [
                "arn:aws:logs:eu-west-1:123456789012:log-group:*:*"
            ]
        }
    ]
}
```

### Cross-account authentication
Is a tricky subject that probably cannot be described here. Broadly speaking the IAM instance role of the host on which the plugin is running
needs to be able to `sts:AssumeRole` the `sts_arn` (and obviously needs `sts_enabled` to be true).

The assumed role should look more-or-less like that above in terms of the actions and resource combinations required.

## Development

After checking out the repo, run `bin/setup` to install dependencies. Then, run `rake spec` to run the tests. You can also run `bin/console` for an interactive prompt that will allow you to experiment.

To install this gem onto your local machine, run `bundle exec rake install`. To release a new version, update the version number in `version.rb`, and then run `bundle exec rake release`, which will create a git tag for the version, push git commits and tags, and push the `.gem` file to [rubygems.org](https://rubygems.org).

## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/sampointer/fluent-plugin-cloudwatch-ingest.

