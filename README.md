# Summary Privacy Issues BigBlueButton

## BigBlueButton always records when used with Greenlight
### Description
When used with the default frontend, greenlight, bbb always creates a recording of rooms, even if the recording button is not pressed.
### Resolution
This is an inherrent feature of BBB. The best resolution so far for me has been to ensure that all recordings that have no recording markers
are deleted directly after the recording ends. Furthermore, all locations where dependent cache files may reside (which are also kept by default)
are mounted on memory filesystems.

## BigBlueButton stores full raw recording data
### Description
BigBlueButton stores the raw recording data for meetings that have recording markers indefinately. This might include parts of the session where BBB was not supposed to record.
### Resolution
Delete raw recording data after a meeting has been archived.

## BigBlueButton does not request consent to a privacy policy when joining a room as a guest.
### Description
BigBlueButton has no feature that forces participants to consent to a privacy policy or being recorded prior to joining a room.
### Resolution
At the moment, I just declare this issue in my privacy policy. For the future, I want to patch GL to request consent before joining.

## All recordings are always accessible when using Greenlight and BigBlueButton together
### Description
While greenlight allows unlisting recordings (the default), this does not mean that the recording is not accessible via its direct link.
### Resolution
Added a third value for the listing setting (public/unlisted/private). Via an additional auth-hook on the recording server, recordings are not displayed when set to private (new default).
Unlisted and Public keep their semantics.

