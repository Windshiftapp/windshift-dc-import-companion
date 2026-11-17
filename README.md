# Windshift Jira DC Import Companion

A small, read-only Jira Data Center plugin that exposes configuration the public
Jira REST API does not, so the Windshift importer can read it faithfully:
workflow graphs with condition and validator descriptors, and screen schemes.

Installing it is optional. Without it, the Windshift Jira importer falls back to
the public designer API and imports fewer details.

## Download

Every tagged release publishes the JAR. The website links to the stable latest
release:

- Stable URL: <https://github.com/Windshiftapp/windshift-dc-import-companion/releases/latest/download/dc-import-companion.jar>
- All versions: <https://github.com/Windshiftapp/windshift-dc-import-companion/releases>

## Install

The companion JAR is unsigned. Jira 11 and later bundle UPM 8, which enables the
app signature check by default, so UPM rejects an unsigned upload until a Jira
administrator allows it. Jira 10 and earlier bundle UPM 7.1, where the check is
off by default.

### Allow the unsigned upload

The check can be disabled for uploaded apps only, or for every app. The first is
narrower and is the recommended option.

**Uploaded apps only.** Add this JVM option and restart Jira:

```
-Datlassian.upm.signature.check.upload.disabled=true
```

Or add this line to `upmconfig/upm.properties` in the Jira home directory:

```
atlassian.upm.signature.check.upload.disabled=true
```

**All apps.** Replace the property name with
`atlassian.upm.signature.check.disabled` in either location.

The `upmconfig` directory defaults to the Jira home directory. Change it with
the `atlassian.upm.configuration.directory` system property.

See Atlassian's [Configuring UPM app signature check](https://confluence.atlassian.com/upm/configuring-upm-app-signature-check-1489470544.html)
for the full property list and [App signing troubleshooting](https://confluence.atlassian.com/upm/app-signing-troubleshooting-1489470589.html)
for the JVM-option and `upm.properties` steps.

### Upload the JAR

Upload the JAR through Jira's Universal Plugin Manager (UPM). The upload needs a
Jira administrator account.

```bash
curl -u admin:admin -X POST \
  -H "X-Atlassian-Token: no-check" \
  -F "plugin=@dc-import-companion.jar" \
  http://localhost:8080/rest/plugins/1.0/
```

The plugin is read-only, requires global `ADMINISTER`, and returns `403`
otherwise. It never returns credentials or secrets.

## Build

Requires JDK 21 and Maven.

```bash
mvn package
# -> target/dc-import-companion-<version>.jar
```

The Atlassian Maven repository is declared in `pom.xml`, so a plain Maven
install is enough. The Atlassian Plugin SDK (`atlas-mvn package`) also works.

## Release

Releases are built and published by GitHub Actions
(`.github/workflows/release.yml`). No manual upload is needed.

1. Bump `<version>` in `pom.xml`.
2. Commit and push the tag:

   ```bash
   git tag v1.1.0
   git push origin v1.1.0
   ```

The workflow verifies that the tag matches the pom version, builds the JAR, and
attaches two assets to the release:

- `dc-import-companion.jar`, a fixed name that backs the stable latest URL.
- `dc-import-companion-<version>.jar`, the versioned artifact.

`workflow_dispatch` runs a build without creating a release.

## API

Base: `/rest/windshift-import/1.0`. Admin-only, read-only, JSON.

### `GET /capabilities`

```json
{ "pluginVersion": "1.0.0", "deploymentType": "datacenter",
  "capabilities": ["workflows", "screens"] }
```

### `GET /project/{projectKey}/screens`

```json
{
  "issueTypeScreens": {
    "default": { "createScreenId": "10002", "editScreenId": "10003", "viewScreenId": "10003" }
  },
  "screens": [ {
    "id": "10002", "name": "Create Screen", "description": "", "tabCount": 1,
    "fields": [ { "id": "summary", "name": "Summary" },
                { "id": "customfield_10104", "name": "Story Points" } ]
  } ]
}
```

### `GET /project/{projectKey}/workflows`

```json
{
  "project": { "id": "10000", "key": "ENG", "name": "Engineering" },
  "issueTypeWorkflowIds": { "10001": "ENG: Software Simplified Workflow" },
  "workflows": [ {
    "id": "ENG: Software Simplified Workflow",
    "name": "ENG: Software Simplified Workflow",
    "statuses": [ { "id": "10000", "name": "To Do" } ],
    "transitions": [ {
      "id": "41", "name": "Done", "type": "DIRECTED",
      "fromStatusIds": ["3"], "toStatusId": "10001",
      "conditions": [ {
        "type": "class",
        "class": "com.atlassian.jira.workflow.condition.PermissionCondition",
        "kind": "permission",
        "args": { "class.name": "...PermissionCondition", "permission": "Resolve Issue" }
      } ],
      "validators": [],
      "postFunctions": [ { "type": "class", "class": "...", "kind": "unknown", "args": {} } ]
    } ]
  } ]
}
```

`kind` is the normalized view of the common built-ins (`user_in_group`,
`user_in_role`, `permission`, `only_assignee`, `subtask_blocking`, `unknown`);
the raw `class` and `args` are always present for forward compatibility.

## Version compatibility

The plugin is built against Jira 11.0.0 (`jira.version` in `pom.xml`). It uses
public Jira APIs and the deprecated-but-still-exported
`com.opensymphony.workflow.loader` package for workflow descriptors. Test
against your target Jira version before rolling it out broadly.

## License

AGPL-3.0, matching the Windshift core. See [LICENSE](LICENSE).
