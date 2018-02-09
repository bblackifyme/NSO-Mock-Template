# NSO-Mock-Template

>This library will become deprecated should the capability to conduct dry-run out format native be exposed via the MAAPI API.


NSO Mock Template object to use for python unittesting service templates to Native device formats

This library seeks to enable to capability to unittest the dry-run native of individual NSO service templates.

This package leverages the ncs_cli capability of running dry-runs against an action.

## Credits and Required NSO Package

This capability is dependent upon Sebastian Strollo's apply-config-template package

https://github.com/NSO-developer/apply-config-template

## Installation

Git clone the repo and pip install:

```shell
git clone https://github.com/bblackifyme/NSO-Mock-Template
cd NSO-Mock-Template
pip install .
```

## Usage

This package can be leveraged inside your Python testing frame work

Example for use in pytest:

```python
from MonkeyTemplate import MonkeyTemplate
import main # NSO service python file with functions to apply config templates
main.ncs.template.Template = MonkeyTemplate

def test_acl_template():
  "unittest to test if an ACL XML generates correct Cisco CLI"
   # Dummy that replicates ncs.application.Template behavior
  mock_template = MonkeyTemplate("service")
  service_variables = ncs.template.Variables()
  service_variables.add('device', "dummy-netsim")
  service_variables.add('acl_name', "acl")
  service_variables.add('p2p_subnet', "10.0.0.0")
  service_variables.add('subnet', "10.0.1.0")
  service_variables.add('lab_wildcard_mask', "0.0.0.255")

  expected_cli = """ip access-list extended aclin
          permit ip 10.0.0.0 0.0.0.3 any
          permit ip 10.0.1.0 0.0.0.255 any
          deny ip any any
         exit
         ip access-list extended aclout
          deny ip 10.0.0.0 0.0.0.3 any
          deny ip 10.0.1.0 0.0.0.255 any
         exit
"""

  generated_cli = main.apply_ipv4_acl("service", service_variables)
  assert expected_cli in generated_cli["native"]['devices'][0]["data"]
```

This is testing the function below:

```python
def apply_ipv4_acl(service, service_variables):
    """Apply the IPv4 ACL Template with service parameters"""
    template = ncs.template.Template(service)
    return template.apply('ipv4-acl', service_variables)
```

With the following XML template:

```XML
<config-template xmlns="http://tail-f.com/ns/config/1.0">
  <devices xmlns="http://tail-f.com/ns/ncs">
    <device>
      <name>{$device}</name>
      <config>
        <ip xmlns="urn:ios">
          <access-list>
            <extended>
              <ext-named-acl>
                <name>{$acl_name}in</name>
                <ext-access-list-rule>
                  <rule>permit ip {$p2p_subnet} 0.0.0.3 any</rule>
                </ext-access-list-rule>
                <ext-access-list-rule>
                  <rule>permit ip {$lab_subnet} {$lab_wildcard_mask} any</rule>
                </ext-access-list-rule>
                <ext-access-list-rule insert="last" >
                  <rule>deny ip any any</rule>
                </ext-access-list-rule>
              </ext-named-acl>
              <ext-named-acl>
                <name>{$acl_name}out</name>
                <ext-access-list-rule>
                  <rule>deny ip {$p2p_subnet} 0.0.0.3 any</rule>
                </ext-access-list-rule>
                <ext-access-list-rule>
                  <rule>deny ip {$lab_subnet} {$lab_wildcard_mask} any</rule>
                </ext-access-list-rule>
              </ext-named-acl>
            </extended>
          </access-list>
        </ip>
      </config>
    </device>
  </devices>
</config-template>
```

## Common issues

### template.apply() returns `{'raw': 'Lorem ipsum', 'native':{'devices':[]}}`

This is an issue where there is something that went wrong in the application of the template.

To see the raw result from NSO print `template.apply()['raw']` this will provide the most info

Common reasons:
  1. There were 'No Modifications to Commit' -> ie the netsim already has the config
  2. A variable was failed to be applied
  3. The device name is mis-typed / nonexistent
  4. NED errors applying the template

### template.apply() mock functionality returns `syntax error: unknown command`

It indicates that the apply-config-template needs to be installed inside NSO.

Available here: https://github.com/NSO-developer/apply-config-template


### template.apply() mock functionality results returns `% No modifications to commit.\n`


It can be an indication on the configuration already be configured on the test device.
Either remove the config, or leverage a 'clean' netsim with no base config.

This can also be an indication of embedded template logic gone astray. If the templates contain complex logic not addressed by the inputs and as a result fails to meet a condition in which the config is applied this will fail.

Also, if the device name key is begining collected via service YANG path, this will occur as well.

For Python services, it is best to get all variables name from the python logic.
