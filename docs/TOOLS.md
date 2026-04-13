# LLM-friendly Lustre Tool CLIs

Short reference for the code-review / bug-tracking CLIs used in
day-to-day Lustre work.  Not installed on the workshop host by
default -- this is a pointer for organizers or attendees who want
to set them up in their own environments after the workshop.

## `jira` -- bug tracking

```
jira get       <LU-XXXXX>
jira search    '<JQL>'
jira comment   <LU-XXXXX> "text"
jira link      <LU-XXXXX> <LU-YYYYY>
jira create    ...
```

Use `-I cloud` for JIRA Cloud; `jira --help` for the full list.

## `gerrit` (`gc`) -- code review

```
gerrit comments        <change>
gerrit maloo           <url>
gerrit info            <change>
gerrit review          <change>
gerrit work-on-patch   <change>
gerrit finish-patch
gerrit watch           <patches.json>
gerrit series-status
```

Run `gerrit --help` for the full command surface.

## Output conventions

All tools emit the data payload by default.  Add `--envelope` for
`{ok, data, meta}`.  Do **not** use `--pretty` (for machine
consumers), and do not pipe through Python.
