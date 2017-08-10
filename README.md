Slack notification sending resource
===================================

Sends messages to [Slack](https://slack.com).

* [Build & promote image](https://ci.starkandwayne.com/teams/main/pipelines/slack-notification-resource) via CI

Resource Type Configuration
---------------------------

```yaml
resource_types:
- name: slack-notification
  type: docker-image
  source:
    repository: cfcommunity/slack-notification-resource
    tag: latest
```

Source Configuration
--------------------

To setup an [Incoming Webhook](https://api.slack.com/incoming-webhooks) go to https://my.slack.com/services/new/incoming-webhook/.

-	`url`: *Required.* The webhook URL as provided by Slack. Usually in the form: `https://hooks.slack.com/services/XXXX`
-   `insecure`: *Optional.* Connect to Concourse insecurely - i.e. skip SSL validation. Defaults to false if not provided.
-   `ca_certs`: *Optional.* An array of objects with the following format:

  ```yaml
  ca_certs:
  - domain: example.com
    cert: |
      -----BEGIN CERTIFICATE-----
      ...
      -----END CERTIFICATE-----
  - domain: 10.244.6.2
    cert: |
      -----BEGIN CERTIFICATE-----
      ...
      -----END CERTIFICATE-----
  ```

  Each entry specifies the x509 CA certificate for the trusted domain.

```yaml
resources:
- name: slack-alert
  type: slack-notification
  source:
    url: https://hooks.slack.com/services/XXXX
```

Behavior
--------

### `out`: Sends message to Slack.

Send message to Slack, with the configured parameters.

#### Parameters

Required:

One or more of the following:
-	`text`: Static text of the message to send.
-	`text_file`: File that contains the message to send. This allows the message to be generated by a
previous task step in the Concourse job.
- `attachments`: An array of one or more message attachments. See the [Slack documentation on message
attachments](https://api.slack.com/docs/message-attachments) for the message structure.
- `attachments_file`: File that contains the json attachments array to send.  This parameter allows
the attachments to be generated by a previous task step in the Concourse job.

If you omit the `text` parameter, the content of the file specified in the
`text_file` parameter will be used verbatim.  Alternatively, you can include
the environment variable `$TEXT_FILE_CONTENT` in the `text` string to include
the content of the file with other static text and interpolated variables (see
Metadata section below)

If you use `text` without using the `$TEXT_FILE_CONTENT` in it, the content
will not be added to your message.

The text and text_file content can contain multiple lines, emojis like :simple_smile:,
and links in form `<http://example.com>` or `<http://example.com|Click here!>`.

If you omit the `attachments` parameter, the contents of the file specified in the
`attachments_file` parameter will be used verbatim.  If the `attachments` parameter
is set, then `attachments_file` will be ignored.  There is no way to use both at the
same time.

Under the philosophy that its better to write something than fail silently,
the following will be sent under these conditions:

- when `text`, `attachments`, `attachments_file` are omitted, and...
  - `text_file` omitted, or present but file missing:
    > *(no notification given)*
  - `text_file` specified and present but file empty:
    > *(missing notification text)*
- when `text` is present and both `attachments` and `attachments_file` are omitted, and `text` evaluates to empty string after variable interpolation:
  > *(missing notification text)*
- when `text` specified with `$TEXT_FILE_CONTENT` and more content, and...
  - `text_file` omitted, or present but file missing
    * `$TEXT_FILE_CONTENT` is replaced with "*(no notification given)*"
  - `text_file` specified and present but file empty:
    * `$TEXT_FILE_CONTENT` is replaced with empty string.

Optional:

-	`channel`: *Optional.* Override channel to send message to. `#channel` and `@user` forms are allowed. You can notify multiple channels separated by whitespace, like `#channel @user`.
-   `channel_file`: *Optional.* File that contains a list of channels to send message to. If `channel` is also specified, the two lists will be concatenated.
-	`username`: *Optional.* Override name of the sender of the message.
-	`icon_url`: *Optional.* Override icon by providing URL to the image.
-	`icon_emoji`: *Optional.* Override icon by providing emoji code (e.g. `:ghost:`).
- `silent`: *Optional.* Do not print curl output (avoids leaking slack webhook URL)
- `always_notify`: *Optional.* Attempt to notify even if there are errors or missing text. (Expects true or false, defaults to false)

Explore formatting with Slack's [Message Builder](https://api.slack.com/docs/formatting/builder).

#### Metadata

Various metadata is available in the form of environment variables. Any environment variables present in the parameters will be automatically evaluated; this enables dynamic parameter content.

The following pipeline config snippet demonstrates how to incorporate the metadata:

```yaml
---
jobs:
- name: some-job
  plan:
  - put: slack-alert
    params:
      channel: '#my_channel'
      text_file: results/message.txt
      text: |
        The build had a result. Check it out at:
        http://my.concourse.url/teams/$BUILD_TEAM_NAME/pipelines/$BUILD_PIPELINE_NAME/jobs/$BUILD_JOB_NAME/builds/$BUILD_NAME
        or at:
        http://my.concourse.url/builds/$BUILD_ID

        Result: $TEXT_FILE_CONTENT
```

See the [official documentation](http://concourse.ci/implementing-resources.html#resource-metadata) for a complete list of available metadata.

Examples
--------

If you're interested in the API of Concourse resources and/or contributing to this resource, you can play with the `out` script using examples. There are some available in the `test` folder.

Note: they have a `params.debug` set so that it only prints out the data structures rather than attempting to invoke the Slack API. Remove it and set a real Slack API to test the script against Slack.

```
> cat test/combined_text_template_and_file_empty.out | ./out .
webhook_url: https://some.url
body: {
  "text": ":some_emoji:<https://my-ci.my-org.com//teams//main//pipelines//jobs//builds/|Alert!>\n",
  "username": "concourse",
  "icon_url": null,
  "icon_emoji": null,
  "channel": null
}

> cat test/combined_text_template_and_file_missing.out | ./out .
webhook_url: https://some.url
body: {
  "text": ":some_emoji:<https://my-ci.my-org.com//teams//main//pipelines//jobs//builds/|Alert!>\n_(no notification provided)_\n",
  "username": "concourse",
  "icon_url": null,
  "icon_emoji": null,
  "channel": null
}

> cat test/combined_text_template_and_file.out | ./out .
webhook_url: https://some.url
body: {
  "text": ":some_emoji:<https://my-ci.my-org.com//teams//main//pipelines//jobs//builds/|Alert!>\nThis text came from sample.txt. It could have been generated by a previous Concourse task.\n\nMultiple lines are allowed.\n",
  "username": "concourse",
  "icon_url": null,
  "icon_emoji": null,
  "channel": null
}

> cat test/text_file_empty.out | ./out .
webhook_url: https://some.url
body: {
  "text": "_(missing notification text)_\n",
  "username": "concourse",
  "icon_url": null,
  "icon_emoji": null,
  "channel": null
}

> cat test/text_file.out | ./out .
webhook_url: https://some.url
body: {
  "text": "This text came from sample.txt. It could have been generated by a previous Concourse task.\n\nMultiple lines are allowed.\n",
  "username": "concourse",
  "icon_url": null,
  "icon_emoji": null,
  "channel": null
}

> cat test/text.out | ./out .
webhook_url: https://some.url
body: {
  "text": "Inline static text\n",
  "username": "concourse",
  "icon_url": null,
  "icon_emoji": null,
  "channel": null
}

```
