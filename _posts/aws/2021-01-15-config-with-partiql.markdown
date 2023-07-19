---
layout: post
title: "Enriching AWS Config Advanced Queries with PartiQL"
categories: aws config
date: 2021-01-15 21:00:00 +0200
tags:
    - aws
    - aws-config
    - partiql
---

# Enriching AWS Config Advanced Queries with PartiQL

Whether you know it or not, you've likely already crossed paths with [PartiQL](https://partiql.org/) and there is a good chance you have already used it. S3 Select uses PartiQL, Redshift Spectrum uses it, and during the Re:Invent 2020 Glue Elastic Views, which allegedly will enable you to combine and replicate data across multiple data stores more easily, was announced with the rumor that it will be based on PartiQL.

What is PartiQL? It is an open source technology developed at AWS and it has been around since [2019](https://aws.amazon.com/blogs/opensource/announcing-partiql-one-query-language-for-all-your-data/). Straight from the website: PartiQL is a query language that provides SQL-compatible query access across multiple data stores containing structured data, semi-structured  data, and nested data. Anyone with headaches cramming nested data schemas into Glue tables or constructing overly complex [jq](https://stedolan.github.io/jq/) queries through trial and error trying to make sense out of JSON files may find some degree of solace switching to PartiQL.

In order to highlight its usefulness I decided to apply it on some AWS Config detective work. During one of my roles we were establishing an overview of resources in our AWS accounts, using a mix of managed and custom Config rules, flagging resources that may contribute to security gaps, unwanted expenditures and more. Up to this day the AWS Config dashboard acts funky and outright refuses to give useful high level numbers. Enter AWS Config Advanced Queries.

## AWS Config Advanced Queries

The advanced queries feature of AWS Config is simple and reminds a lot of S3 Select, you can select resources by their type, name and so on using some pruned SQL syntax. You even get an array with compliance verdicts for whatever rules apply to the resource. Sounds easy enough to crunch those earlier mentioned numbers? Think again.

Surreptitiously hidden in the AWS documentation we bump into [annoyances](https://docs.aws.amazon.com/config/latest/developerguide/querying-AWS-resources.html#query). If the lack of DISTINCT, JOINS SQL syntax isn't already a red flag then despair for this next situation. Let's say you have an IAM user and two Config rules are active. One is named "AccessKeysRule" and checks whether the access keys were used within the last 90 days and another named "MFAEnabled" to check whether MFA has been enabled. Let's say for one user MFA is enabled (compliant) but the access keys have been idle for longer than 90 days (non-compliant). This next query will return "incorrectly" that IAM user:

{% highlight sql %}
SELECT
	configuration
WHERE
	configuration.configRuleList.complianceType = 'NON_COMPLIANT' AND
	configuration.configRuleList.configRuleName = 'MFAEnabled'
{% endhighlight %}

And so will this one:

{% highlight sql %}
SELECT
	configuration
WHERE
	configuration.configRuleList.complianceType = 'COMPLIANT' AND
	configuration.configRuleList.configRuleName = 'AccessKeysRule'
{% endhighlight %}

The reason is that the query will simply iterate over all rules and treat the compliance type and rule name separate. If there is a non-compliant rule and there is a rule with the name "MFAEnabled", than we have a match, it will not limit the compliance status check to just the "MFAEnabled" rule. Same applies for the "AccessKeysRule" and the compliant constraint. A lack of expressiveness is blocking our attempt to get the numbers we want. Enter PartiQL.

## (P)Arti(QL)ulating your AWS resources

Instead of tackling the problem in AWS, we use the CLI to get relevant data to our side and then we unleash the expressive freedom of PartiQL to get the numbers we have been looking for. With a Bash script we supply an advanced query and paginate over the results storing them locally:

{% highlight bash %}
#!/bin/bash
set -xe

QUERY=$1
OUTPUT_FILE="./config-$(date +%s).jsonl"
CONFIG_FILE="./config-$(date +%s).env"

function paginate() {
  if [ -z $1 ]
  then
    output=$(aws configservice select-resource-config --expression "$QUERY")
  else
    output=$(aws configservice select-resource-config --expression "$QUERY" --next-token $1)
  fi

  echo $output | jq -r '.Results[]' >> $OUTPUT_FILE
  echo $output | jq -r '.NextToken'
}

while [ "$NEXT_TOKEN" != "null" ]
do
  NEXT_TOKEN=$(paginate $NEXT_TOKEN)
done

echo "{'data': read_file('$OUTPUT_FILE')}" > $CONFIG_FILE
{% endhighlight %}

With this Bash script in place, let's pool all the IAM users with

{% highlight bash %}
./collect.sh "SELECT configuration WHERE configuration.targetResourceType = 'AWS::IAM::User'"
{% endhighlight %}

Two files will be created, one containing raw data in JSON lines format, the other one reading in the former and presenting it in PartiQL as the data table. Install PartiQL so we can start sifting through data

{% highlight bash %}
rlwrap partiql -e config-<...>.env
{% endhighlight %}

The rlwrap package is worth installing to lookup history and scrolling through your query history in PartiQL. Install it with your Linux package manager or resort to Homebrew on Mac. With everything in place, perform a quick count:

{% highlight sql %}
SELECT count(*) FROM data
{% endhighlight %}

If everything is set up fine, you will now know the number of IAM users in your account. Time to crunch numbers.

## Querying Nested Data

With PartiQL you can join records with nested data within the record, similar to a data explode. Using this mechanism we can filter all resources for their compliance to a particular rule, which we could not do before:

{% highlight sql %}
SELECT
    d.targetResourceId,
    r.configRuleName,
    r.complianceType
FROM data d, d.configuration.configRuleList r
WHERE
    r.configRuleName = 'MFAEnabled'
{% endhighlight %}

Easy as that. For those gettings hands on with the blog post, you might want some extra data, since the resource IDs for IAM users are not as intuitive as the names. Insight tip into AWS Config advanced queries: the data we pulled are all resources of the type `AWS::Config::ResourceCompliance`. Let's also pull more data using the earlier scripts for IAM users:

{% highlight bash %}
./collect.sh "SELECT * WHERE resourceType = 'AWS::IAM::User'"
{% endhighlight %}

If you open the env file you find the structure is rather simple:

{% highlight bash %}
{
    'data': read_file('./config-<...>.jsonl')
}
{% endhighlight %}

Let modify this file to include the JSON lines from the first query as well where we pulled the compliance results:

{% highlight bash %}
{
    'resources': read_file('./config-<...>.jsonl'),
    'results': read_file('./config-<...>.jsonl'),
}
{% endhighlight %}

We will now have two tables available when we start up PartiQL, one with the resources, one with the compliance results. Putting the pieces together:

{% highlight ts %}
SELECT
    r1.resourceName,
    r3.complianceType
FROM
    resources r1
JOIN
    results r2
ON r1.resourceId = r2.configuration.targetResourceId,
r2.configuration.configRuleList r3
WHERE r3.configRuleName = 'MFAEnabled'
{% endhighlight %}

This will list the IAM users by their name and their compliance results for our specific rule of interest. From this point on you may wish to run analyses using [aggregators](https://docs.aws.amazon.com/config/latest/developerguide/aggregate-data.html) over entire AWS accounts or even organizations (and adapt the Bash script using the [correct CLI command](https://docs.aws.amazon.com/cli/latest/reference/configservice/select-aggregate-resource-config.html)), or include [relationships](https://docs.aws.amazon.com/config/latest/developerguide/examplerelationshipqueries.html) between AWS resources to perform more fine grained aggregations in PartiQL.

We showed how PartiQL is an easy enough tool to circumvent the shortcomings of the not so advanced queries in AWS Config and how its simple syntax is able to bring together the benefits of SQL with the nested data structures. If you want to dig deeper, have a look at this [video](https://www.youtube.com/watch?v=ZsEOhCOFOe4) from AWS Re:Invent 2019 showcasing PartiQL. Also take a look at the specification [here](https://partiql.org/assets/PartiQL-Specification.pdf).
