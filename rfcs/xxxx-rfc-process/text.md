# RFC Process <img style="float: right; width: 20px" src="./imgs/monkey_mascot.jpg">

**Authors:** [Arathi Mani](mailto:arathi.mani@chanzuckerberg.com)

**Approvers:** [Timmy Huang](mailto:thuang@chanzuckerberg.com), [Marcus Kinsella](mkinsella@chanzuckerberg.com), [Brian Raymor](mailto:braymor@chanzuckerberg.com)

**Reviewers:** [Madison Dunitz](madison.dunitz@chanzuckerberg.com), [Brian McCandless](bmccandless@chanzuckerberg.com), [Trent Smith](trent.smith@chanzuckerberg.com), [Colin Megill](colin.megill@chanzuckerberg.com), [Seve Badajoz](sbadajoz@chanzuckerberg.com)

## tl;dr 

This RFC contains the details about the RFC process for all cellxgene related designs.


## Background

Previously the "RFC" (a.k.a. "design doc review") process was executed on Google Docs. There were two problems

1. Updates were quite tricky to make. Do you update the doc directly? Do you have a changelog? which version is latest? Do you need to apply the change in the changelog in the body of the doc? 
2. Designs decisions were not easily accessibly by our external community. How does the external community, with whom we have a close relationship, keep track of our decisions and progress if the designs are hidden in a private Google Drive?

In order to alleviate these questions and concerns, we have moved to Github to keep track of our RFCs moving forward.


## How to create a new RFC

The following are the steps to writing and approving a new RFC (not modifying):

1. Create a new branch of the `cellxgene-rfcs` repo.
2. Create a new directory under the `rfcs` directory named `XXXX-name-of-rfc`, where `name-of-rfc` is the name of your RFC, not those words directly.
3. Copy over the contents of the `templates` directory into your newly created directory. This contains a folder for images and the RFC template.
4. Rename `rfc_template.md` to `text.md` and edit it. Add images that you'd like to include in the `imgs` folder.
5. When you are ready for review, create a pull request. Add your approvers (and any other people you'd like to review your work) as reviewers on the PR. 
6. As a courtesy, post a heads up on the [#org-sci-eng-single-cell](https://chanzuckerbergteam.slack.com/archives/GQGPP7925) Slack channel that there is a design available for review, specifically calling out the approvers. Generally, two weeks is nice for substantial designs to get reviewed, but use your best judgement on how much time you'd like for your design to be under review.
7. Once all comments are addressed and the *approvers* have approved your pull request, **change the name of your directory** from `XXXX-name-of-rfc` to pick the next number (for example, this one will be `0000-rfc-process`).
8. Push your design! Delete your branch! Close your design ticket! Code away!


## How to update an existing RFC

The following are the steps to revising an existing RFC:

1. Create a new branch of the `cellxgene-rfcs` repo.
2. Make your changes in the appropriate RFC. If there are new approvers, **append** them to the list of approvers at the top of the RFC.
3. Create a pull request. This is probably the most important step and most different step from creating a branch new RFC. In the top level comment of the pull request, ensure that you succinctly summarize the changes. Also note in the top level comment of the RFC who the approvers are of the specific pull request. This is done so that later in `blame` you can easily find out the commits that affected a file (RFC) and who was involved.
4. As a courtesy, post a heads up on the [#org-sci-eng-single-cell](https://chanzuckerbergteam.slack.com/archives/GQGPP7925) Slack channel that there is a design modification available for review, specifically calling out the approvers. Use your best judgement on how much time you'd like for your modification to be under review, but roughly should be anywhere from a couple days to 2 weeks, depending on the size of the change. Note that huge changes like re-architecting probably warrants its own RFC.
5. Once all comments are addressed and the *approvers* have approved your pull request, push the change!
6. Delete your branch.


## Alternatives

This process was initially done on Google Docs. As mentioned earlier, the process didn't work out well for keeping track of modifications to the RFC as versioning was not easy to use. Other ideas that was considered include [IETF's Github process](https://tools.ietf.org/html/draft-ietf-git-using-github-06) as well as the [Data Coordination Platform RFC process](https://github.com/HumanCellAtlas/dcp-community/blob/master/rfcs/text/0001-rfc-process.md).

## References

[0] [IETF's Github Process](https://tools.ietf.org/html/draft-ietf-git-using-github-06)

[1] [Google Developer Documentation Style Guide](https://developers.google.com/style/highlights#introduction_1)

[2] [Data Coordination Platform RFC Process](https://github.com/HumanCellAtlas/dcp-community/blob/master/rfcs/text/0001-rfc-process.md)
