---
authors: Andrej Tokarčík (andrej@goteleport.com)
state: draft
---

# RFD 16 - Dynamic Configuration

## What

Some resources like `services.AuthPreference` or `services.ClusterConfig` are
live representations of certain parts of the `auth_service` configuration
section.  These resources can be understood to constitute *dynamic*
configuration of an auth server, as opposed to *static* configuration defined
by its `teleport.yaml` file.

The current behaviour is that Teleport creates dynamic configuration
implicitly by deriving it from static configuration during auth server
initialization. The recent proposals of exposing the related resources via
`tctl` would however allow the dynamic configuration to be created/updated
explicitly by the user, independently of static configuration.

This RFD presents several possible scenarios involving the interaction of
explicitly-managed dynamic and static configuration, including the respective
design decisions to be made, with the goal of reaching a consensus on the
configuration workflows to be supported by Teleport.

## Why

1. Automated/programmatic management of Teleport clusters.

2. Teleport-as-a-service offerings that do not allow direct access
   to `teleport.yaml` files.

## Scenarios

We use the example of the `authentication` subsection of `auth_service`
in `teleport.yaml` (as "the" static configuration) and the corresponding
resource `services.AuthPreference` (as "the" dynamic configuration).
The latter is assumed to be manipulable via `tctl` using the identifier `cap`.

### Scenario 1: static configuration specified

1. The `authentication` section is specified in `teleport.yaml`.
   The backend can have any `AuthPreference` data stored from the last
   auth server run.

2. The auth server initialization procedure MUST commit data derived from the
   static configuration to the backend, overwriting any potential resource data
   already stored there.

3. `tctl get cap` returns the resource data in line with `teleport.yaml`.

Therefore, static configuration is always given precedence.  This should also
ensure backward compatibility with the already established workflows.

### Scenario 2: static never specified & dynamic not updated by user

1. The `authentication` section has never been specified in `teleport.yaml`.
   No user changes have been made to the dynamic configuration.

2. The auth server initialization procedure MAY or MAY NOT commit the default
   settings to the backend (in the latter case a structure holding the default
   settings is returned in place of the missing resource when the resource
   is requested).

#### Option 2.A

In this option, dynamic-configuration resources are understood to exist only
if they have been committed as a result of having been specified in static
configuration or via `tctl create`.

3. The command `tctl get cap` would therefore return an error saying
   "authentication preference not found".  To view the default state, `tctl
   get` would have to be invoked with a new CLI flag.

#### Option 2.B

3. `tctl get cap` returns the resource data corresponding to the default
   settings.

### Scenario 3: static reverted to unspecified & dynamic not updated by user

1. The `authentication` section used to be specified in `teleport.yaml` but
   it has been removed and left unspecified since the last auth server init.
   No user changes have been made to the dynamic configuration.

#### Option 3.A

2. The auth server initialization procedure retains the last stored dynamic
   configuration which had been derived from the last specified static
   configuration.  In particular, the backend MUST NOT be overwritten
   with the default settings.

3. `tctl get cap` returns the resource data corresponding to the last specified
   static configuration.

#### Option 3.B

2. The auth server initialization procedure MUST NOT retain the last stored
   dynamic configuration.  The resource data are either overwritten with the
   default settings or deleted (in the latter case a structure holding the
   default settings is returned in place of the missing resource when the
   resource is requested).

3. `tctl get cap` then behaves as in Scenario 2.

### Scenario 4: static never specified & first `tctl create`

1. The `authentication` section has never been specified in `teleport.yaml`.
   No user changes have been made to the dynamic configuration.

2. A user issues the command `tctl create authpref.yaml` with valid
   authentication preferences stored in the YAML file.
   (Issuing `tctl create -f authpref.yaml` would have the same effect.)

3. The dynamic configuration is updated: the resource data read from
   `authpref.yaml` MUST be written to the backend (potentially overwriting
   the default data already stored there).

4. The command returns with success.

5. Upon the next auth server restart with `authentication` still unspecified
   in `teleport.yaml`, the auth server initialization procedure retains
   the last stored dynamic configuration, i.e. the one from `authpref.yaml`.

6. `tctl get cap` returns the resource data corresponding to `authpref.yaml`.

### Scenario 5: static never specified & repeated `tctl create`

1. The `authentication` section has never been specified in `teleport.yaml`.
   A user has already made explicit changes to the dynamic configuration
   using `tctl create`.

2. A user issues the command `tctl create authpref.yaml` with valid
   authentication preferences stored in the YAML file.

#### Option 5.A

3. The command is rejected with an "already exists" error, recommending
   to use the `-f`/`--force` flag to overwrite the already existing
   `AuthPreference` resource.

#### Option 5.B

3. The command is accepted and returns with success.

### Scenario 6: static specified & `tctl create` attempt

1. The `authentication` section is specified in `teleport.yaml`.
   The auth server is running with dynamic configuration derived from static
   configuration as in Scenario 1.

2. `tctl create authpref.yaml` returns with an error indicating that the
   respective static configuration has been specified and that its derived
   dynamic configuration is fixed.

**Needs decision:** There are differing opinions on whether the following
should be allowed or it might constitute a security risk.

3. `tctl create -f authpref.yaml` allows the user to overwrite the dynamic
   configuration but only temporarily until it gets replaced by the
   static configuration as in Scenario 1.

## Configuration source preferred by auth server init

The following table summarily depicts a key aspect of the scenarios elaborated
above, namely the question of which configuration source is to be preferred by the
auth server initialization procedure:

|                        | **dynamic updated by user** | **dynamic not updated by user** |
|          :---:         |            :---:            |              :---:              |
|  **static specified**  |            static           |              static             |
| **static unspecified** |           dynamic           |  defaults OR last static [???]  |

(The state marked [???] is unclear and needs decision, see Scenario 3.)