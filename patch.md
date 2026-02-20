# patch.md

## meta

- name: stalwart
- target: Stalwart Mail Server Rust fork
- version: 0.16.27

## commits

### 1. Use listener heuristics instead of local_port to identify smtp connections

#### background

Stalwart Mail Server configures default MTA stage expressions (`session.ehlo`, `session.auth`,
and `session.data.add-headers`) to differentiate between inbound relay SMTP (port 25) and client
submission (ports 587/465). Upstream defaults hardcode `local_port == 25` and `local_port != 25`
expressions to detect these traffic classes.

In containerized environments (such as Docker, Kubernetes, or Nomad), port forwarding setups,
HAProxy/reverse proxy termination, or deployments binding Stalwart to non-standard ports (e.g. 2525),
evaluating `local_port` breaks. Inbound mail delivered to non-25 container ports is erroneously treated
as client submission—requiring authentication and skipping essential inbound headers (`Received`,
`Authentication-Results`, `Received-SPF`, `Return-Path`, `Message-ID`, `Date`).

This change replaces `local_port == 25` and `local_port != 25` checks across the default MTA stage
expression definitions with `listener == 'smtp'` and `listener != 'smtp'`. This allows Stalwart to
correctly classify SMTP connections by configured listener identifier rather than local socket port
number, enabling container port mapping and custom listener ports without breaking default MTA behavior.

#### files

##### crates/registry/src/schema/structs_impl.rs (modify)

Replace `local_port == 25` with `listener == 'smtp'` and `local_port != 25` with `listener != 'smtp'`
in default expressions for `MtaStageAuth` (`mechanisms`, `require`), `MtaStageData` (`add_received_header`,
`add_received_spf_header`, `add_return_path_header`, `add_auth_results_header`, `add_message_id_header`,
`add_date_header`), and `MtaStageEhlo` (`reject_non_fqdn`).

#### verify

- `cargo check -p registry` compiles successfully.
- `cargo test -p registry` passes.
- Default expressions generated for MTA auth, data headers, and EHLO validation use `listener == 'smtp'` / `listener != 'smtp'` instead of `local_port == 25`.
- SMTP traffic received over non-standard ports (e.g., mapped port 2525 on a listener named `smtp`) correctly identifies as inbound SMTP.
