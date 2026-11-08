# GitHub Issue Edits

## MANDATORY: Fetch the live body before editing

Before calling `gh issue edit --body` / `--body-file`, always fetch the current body first with
`gh issue view <n> --json body --jq .body > /tmp/issue_<n>_body.md`. Apply minimal, surgical edits to that freshly
fetched file, then push it back. Never push a body composed from an earlier in-conversation snapshot.

The user pastes screenshots and comments directly into issues between turns. A stale local body will silently wipe those
additions. Commenting on an issue (`gh issue comment`) is the exception, not the rule — most updates belong in the body.

## MANDATORY: Issue bodies open with an executive summary

Every issue body starts with an `## Executive summary` heading, and that heading comes before any other heading,
screenshot, code block or list. Under it sits a single paragraph of at most three sentences.

Someone who reads the title and that paragraph, and never scrolls, should understand what is wrong, who it hurts and
what needs to happen. Write it for a reader who does not know the code: no class names, no method names, no file paths,
no framework names. Cover the problem, its effect on users or on the business, and the consequence of leaving it alone
or the change being asked for.

Do not restate the title. The title names the thing, the executive summary explains why anyone should care.

Details, reproduction steps, screenshots, measurements and acceptance criteria follow under their own headings below,
and that is where the technical vocabulary belongs.

This applies to issues you create and to issues you rewrite. Do not retrofit an executive summary onto an existing issue
that lacks one unless the user asks for it, since that conflicts with the surgical-edit rule above.

Example, for an issue titled "Export button is disabled for large reports":

    ## Executive summary

    Users cannot export reports larger than a fixed size, so they copy the data by hand and make mistakes. The limit
    comes from a timeout in the export request, not from the data itself. Raising the timeout or moving the export
    to a background job fixes it; until then, large reports cannot be exported at all.

## MANDATORY: Record deliberate deviations from an issue on the issue

When a plan or an implementation deliberately deviates from an issue's acceptance criteria, a scope item is deferred,
a criterion is dropped, or the agreed behaviour differs from what the body asks for, the deviation is recorded on the
issue itself. Either edit the affected acceptance criterion in the body (surgically, per the fetch rule above) or, when
the body should stand as originally written, comment on the issue.

The issue body is the contract whoever picks the work up reads. If the deviation lives only in a plan document or in
the code, the issue and the implementation quietly disagree, and the next reader rebuilds the missing half from the
issue text alone. For example: an issue asks for a setting to be editable by two roles, the work ships for one role
only with the other half deferred, and without a note on the issue that second half gets built again from the body by
the next person.
