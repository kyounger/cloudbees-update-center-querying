## My notes on how to query the CloudBees Update Centers

These snippets include some extensible code that helps query the UpdateCenter for plugins in various ways. It uses `jq` and [`dsq`](https://github.com/multiprocessio/dsq). Note the fairly plain sql statements used by `dsq` that could be more complicated, if desired.

The first section of this code uses wget (you can other tools, e.g. curl) to pull the UC. However, the UC is in JSONP and a quick way to handle that is to strip the first and last line of the file. This effectively converts it to proper json.

``` sh
export ENVELOPE=envelope-cje
export VERSION=2.303.3.3
wget -q -O - "https://jenkins-updates.cloudbees.com/update-center/$ENVELOPE/update-center.json?version=$VERSION" | sed '1d' | sed '$d' > uc.json

jq '[.plugins | .[] | {buildDate, gav, groupId, name, releaseTimestamp, requiredCore, sha1, title, url, version}]' uc.json \
  | dsq -s json 'select name, version from {}' \
  | jq -r '(map(keys) | add | unique) as $cols | map(. as $row | $cols | map($row[.])) as $rows | $cols, $rows[] | @csv'
```

This snippet (which uses the same `uc.json` generated above), shows all plugins in that UC that have a dependency on the popular `github-branch-source` plugin.

``` sh
jq '[.plugins[] | {name: .name, dep: .dependencies[].name }  ]' uc.json \
  | dsq -s json 'select name, dep from {} where dep = "github-branch-source"'
```

Note that the ENVELOPE variable is product- _and_ component-specific.

For CloudBees CI Modern, the envelopes for Operations Center and Managed Controllers are `envelope-core-oc` and `envelope-core-mm`, respectively.
For the legacy CJP, the envelopes for Operations Center and Managed Controllers are `envelope-cjoc` and `envelope-cje`, respectively.
