# Contributing Guidelines

The following is a set of guidelines for contributing to nginx project.
We really appreciate that you are considering contributing!

## Table of Contents

- [Ask a Question](#ask-a-question)
- [Report a Bug](#report-a-bug)
- [Suggest a Feature or Enhancement](#suggest-a-feature-or-enhancement)
- [Open a Discussion](#open-a-discussion)
- [Submit a Pull Request](#submit-a-pull-request)
- [Issue Lifecycle](#issue-lifecycle)

## Ask a Question

To ask a question, open an issue on GitHub with the label `question`.

## Report a Bug

To report a bug, open an issue on GitHub with the label `bug` using the
available bug report issue template. Before reporting a bug, make sure the
issue has not already been reported.

## Suggest a Feature or Enhancement

To suggest a feature or enhancement, open an issue on GitHub with the label
`feature` or `enhancement` using the available feature request issue template.
Please ensure the feature or enhancement has not already been suggested.

## Open a Discussion

If you want to engage in a conversation with the community and maintainers,
we encourage you to use
[GitHub Discussions](https://github.com/nginx/nginx/discussions).

## Submit a Pull Request

Follow this plan to contribute a change to NGINX source code:

- Fork the NGINX repository
- Create a branch
- Implement your changes in this branch
- Submit a pull request (PR) when your changes are tested and ready for review

Refer to
[NGINX Development Guide](https://nginx.org/en/docs/dev/development_guide.html)
for questions about NGINX programming.

### Formatting Changes

- Changes should be formatted according to the
[code style](https://nginx.org/en/docs/dev/development_guide.html#code_style)
used by NGINX; sometimes, there is no clear rule, in which case examine how
existing NGINX sources are formatted and mimic this style; changes will more
likely be accepted if style corresponds to the surrounding code

- Keep a clean, concise and meaningful commit history on your branch, rebasing
locally and breaking changes logically into commits before submitting a PR

- Each commit message should have a single-line subject line followed by verbose
description after an empty line

- Limit the subject line to 67 characters, and the rest of the commit message
to 76 characters

- Use subject line prefixes for commits that affect a specific portion of the
code; examples include "Upstream:", "QUIC:", or "Core:"; see the commit history
to get an idea of the prefixes used

- Reference issues in the the subject line; if the commit fixes an issue,
[name it](https://docs.github.com/en/issues/tracking-your-work-with-issues/linking-a-pull-request-to-an-issue)
accordingly

### HTTP Status Code Guidelines

- Contributors should use the centralized status code API (`ngx_http_status_set`)
instead of direct assignments to `r->headers_out.status`

- The new API provides RFC 9110 compliance validation and consistent error
handling across all HTTP modules

- Use the following pattern when setting HTTP status codes:

```c
/* OLD approach (deprecated but still functional): */
r->headers_out.status = NGX_HTTP_NOT_FOUND;

/* NEW approach (recommended): */
ngx_int_t rc = ngx_http_status_set(r, NGX_HTTP_NOT_FOUND);
if (rc != NGX_OK) {
    ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                  "failed to set HTTP status");
    return NGX_HTTP_INTERNAL_SERVER_ERROR;
}
```

- Direct status assignments are deprecated but still functional for backward
compatibility with existing code and third-party modules

- Contributors should include proper error handling when using the status code
API, as shown in the example above; validation failures should be logged and
handled gracefully with appropriate fallback status codes

- For detailed migration guidance and API documentation, refer to
`docs/migration/status_code_api.md`

- All new code contributions must use the centralized status code API
exclusively; pull requests that introduce new direct status assignments will
be requested to adopt the API before acceptance

### Before Submitting

- The proposed changes should work properly on a wide range of
[supported platforms](https://nginx.org/en/index.html#tested_os_and_platforms)

- Try to make it clear why the suggested change is needed, and provide a use
case, if possible

- Passing your changes through the test suite is a good way to ensure that they
do not cause a regression; the repository with tests can be cloned with the
following command:

```bash
git clone https://github.com/nginx/nginx-tests.git
```

- Submitting a change implies granting project a permission to use it under the
[BSD-2-Clause license](https://github.com/nginx/nginx/blob/master/LICENSE)

## Issue Lifecycle

To ensure a balance between work carried out by the NGINX engineering team
while encouraging community involvement on this project, we use the following
issue lifecycle:

- A new issue is created by a community member

- An owner on the NGINX engineering team is assigned to the issue; this
owner shepherds the issue through the subsequent stages in the issue lifecycle

- The owner assigns one or more
[labels](https://github.com/nginx/nginx/issues/labels) to the issue

- The owner, in collaboration with the wider team (product management and
engineering), determines what milestone to attach to an issue;
generally, milestones correspond to product releases
