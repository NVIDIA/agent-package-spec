# Contributing

Thank you for helping improve the Agent Package Specification.

## Start with an issue

Open an issue for technical feedback, interoperability experience, ambiguity,
or a proposed design change. Explain the use case and the problem before
proposing detailed wording. Small editorial corrections may go directly to a
pull request.

## Propose a change

Keep each pull request focused on one concern. A specification change should
include:

- the problem or use case;
- the proposed behavior or wording;
- compatibility implications;
- security and interoperability implications; and
- updates to examples affected by the change.

Normative requirements use the key words defined by RFC 2119 and RFC 8174 only
when those words appear in uppercase.

## Signing Off Your Work

- We require that all contributors "sign-off" on their commits. This certifies
  that the contribution is your original work, or you have rights to submit it
  under the same license, or a compatible license.

  - Any contribution which contains commits that are not Signed-Off will not be
    accepted.

- To sign off on a commit you simply use the `--signoff` (or `-s`) option when
  committing your changes:

  ```bash
  $ git commit -s -m "Add cool feature."
  ```

  This will append the following to your commit message:

  ```text
  Signed-off-by: Your Name <your@email.com>
  ```

- Full text of the DCO (<https://developercertificate.org/>):

  ```text
  Developer Certificate of Origin
  Version 1.1

  Copyright (C) 2004, 2006 The Linux Foundation and its contributors.

  Everyone is permitted to copy and distribute verbatim copies of this
  license document, but changing it is not allowed.


  Developer's Certificate of Origin 1.1

  By making a contribution to this project, I certify that:

  (a) The contribution was created in whole or in part by me and I
      have the right to submit it under the open source license
      indicated in the file; or

  (b) The contribution is based upon previous work that, to the best
      of my knowledge, is covered under an appropriate open source
      license and I have the right under that license to submit that
      work with modifications, whether created in whole or in part
      by me, under the same open source license (unless I am
      permitted to submit under a different license), as indicated
      in the file; or

  (c) The contribution was provided directly to me by some other
      person who certified (a), (b) or (c) and I have not modified
      it.

  (d) I understand and agree that this project and the contribution
      are public and that a record of the contribution (including all
      personal information I submit with it, including my sign-off) is
      maintained indefinitely and may be redistributed consistent with
      this project or the open source license(s) involved.
  ```

## Public-content requirements

Contributions must be suitable for a public repository. Do not include
credentials, confidential information, customer information, private URLs,
internal ticket identifiers, or unpublished implementation details. Use
public, generic examples when organization-specific details are unnecessary.

## Review

Maintainers may request changes, split a proposal into smaller decisions, or
decline changes that are outside the scope of this specification. Acceptance
means inclusion in the current draft; it does not by itself make the proposal
an adopted industry standard.
