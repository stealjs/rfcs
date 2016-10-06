# Steal RFCs

A place to discuss project-level changes to Steal and request new features that don't fit in any particular repository.

## What qualifies

Things that qualify for RFCs include:

* A new official plugin.
* A feature that results in a breaking change (a major semver bump is required).
* A change to underlying Steal code (like the SystemJS backend) that would result in large code changes.

*Note* you don't need permission to create a plugin. Just go create it and publish it yourself. If you want the plugin to be under the stealjs organization, please create an RFC. If you don't want to create the plugin yourself, create an RFC.

## Process

1. File an issue to gauge interest. If you plan on working on the feature yourself state that in the issue. Otherwise it will be given a [needs hero](https://github.com/stealjs/rfcs/issues?q=is%3Aopen+is%3Aissue+label%3A%22needs+hero%22) label. If it's a controversial issue you might wait some time before proceeding to the next step to see if there is positive feedback.
2. Copy `template.md` to `text/my-feature.md` where "my-feature" is a short description of the change.
3. Fill out each section of the form. Try to be as concise as possible.
4. Submit a pull request. As a pull request the RFC will receive feedback from community and core team members. You may be asked to revise sections of the RFC in response to feedback.
5. Build consensus. The hard part of any large change in open-source is convincing others it is worthy. Changes with wide consensus are much more likely to be accepted.
6. RFCs that are a candidate for inclusion will enter a final comment period lasting 7 days. This will give people a final chance to make their cases.
7. Once the 7 days have passed, the core team may merge the pull request making the RFC approved. **Once approved the feature may begin being worked on**.

## Post-RFC

Once an RFC is approved and merged into the `text/` folder, that doesn't guarantee the feature will be implemented. It only says that:

* Someone has taken on responsibility for implementation.
* There is interest in the community for the feature, so it is likely to be accepted.

At this point the feature still needs to be implemented by the hero and submitted to the proper repository. Time and implementation details could still prevent a feature from being merged.

## Credit

I largely stole the process from [Ember RFCs](https://github.com/emberjs/rfcs).
