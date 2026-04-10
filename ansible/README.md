```text
app-epicbook (Repository Root)
├── azure-pipelines.yaml          <-- The pipeline we just wrote
├── .gitignore                    <-- Ignores .vault_pass, etc.
└── ansible/                      <-- Everything Ansible stays here
    ├── inventory.ini             <-- (Move this inside /ansible)
    ├── site.yml                  <-- (Move this inside /ansible)
    ├── group_vars/
    │   └── web/
    │       ├── vars.yml
    │       └── vault.yml
    └── roles/
        ├── common/
        │   ├── handlers/
        │   └── tasks/
        ├── epicbook/
        │   ├── tasks/
        │   └── templates/
        └── nginx/
            ├── handlers/
            ├── tasks/
            └── templates/
```