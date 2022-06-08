# RFC

This project contains the RFCs for the skyscraper project.

## When you need to follow this process

You need to follow this process if you intend to make design and operational changes to the skyscraper project. While
not every PR requires an RFC, we ask that you make an RFC in situations such as the following:

- Project governance changes
- Introduction or discontinuance of a feature.
- Change that may result in breaking changes between versions.
- Ideas for architectural improvements

## Process Steps

1. Clone this repository.
2. Copy `0000-template.md` to `rfc/0000-my-feature.md` (where "my-feature" is a descriptive name. Please don't assign an RFC number yet).
3. Fill in the RFC document.
4. Submit a pull request (please name the branch the same as the corresponding RFC).
   1. As a pull request, the RFC will receive design feedback from the Team, and the author should be prepared to revise it in response.
5. The author of the pull request should build consensus and integrate feedback.
6. Eventually, the Team will decide whether to merge the RFC.
   1. An RFC may be rejected by the Team after discussion has settled and comments have been made summarizing the rationale for rejection. A member of the Team should then close the RFC's associated merge request.
   2. An RFC may be accepted by the Team. If so, the team should:
      1. Assign an RFC number and change the file name to reflect it.
      2. Merge the RFC's associated merge request
   3. After the RFC is merged, it becomes "active" and the team can pick it up for implementation. 

## The RFC life-cycle

Once an RFC becomes "active", the authors may begin to plan the change. Becoming "active" is not a rubber-stamp, and does not mean the change will ultimately be made. It's a design document that the team has agreed on and may implement at its own priority.

Modifications to active RFC's can be done in followup pull requests. It's only natural that alternatives can arise or plans can change with information found during an implementation.

In the end, the RFC should reflect the current consensus and plan.
