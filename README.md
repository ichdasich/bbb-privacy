# BigBlueButton

## Recordings
### BigBlueButton always records when recording of a room is enabled
When a room is created in BBB that allows recordings (i.e., the recording button is present) BBB will always record the session.
This is independent of the button actually being pressed. 
The technical reason behind this is that parts of the recordings (esp. the SVG files for the Whiteboard) depend on earlier state to be properly processed, see: https://docs.bigbluebutton.org/dev/recording.html
By default these files are stored for two weeks (see 'Retention of Cache Files' below).
Furthermore, depending on the use-case and jurisdiction it might be prudent to retain the option to create 'retroactive' recordings, e.g., when users forgot to click the recording button.

#### Resolution
Operators have two options for handling this:

##### Globally disable recordings in BBB
Users can change `/usr/share/bbb-web/WEB-INF/classes/bigbluebutton.properties` and adjust `disableRecordingDefault=false` to `disableRecordingDefault=true` to globally disable recordings.

##### Deploy a post-recording script that removes recordings without recording markers
Users can deploy a custom script that purges recordings and cache-files of recordings if no recording markers are present. 

Simple version:
`/etc/sudoers:`
`bigbluebutton ALL = NOPASSWD: /usr/bin/bbb-record`

and 
`/usr/local/bigbluebutton/core/scripts/archive/archive.rb (after line 242):`
`	  BigBlueButton.logger.info("There's no recording marks for #{meeting_id}, deleting the recording.")`
`	  system('sudo', 'bbb-record', '--delete', "#{meeting_id}") || raise('Failed to delete local recording')`

For a more complete version that also explicitly deletes cache files of recordings for freeswitch/kurento, please see: https://github.com/Kalagon/bbb-recording-archive-workaround 

### BigBlueButton stores full raw recording data
#### Description
BigBlueButton stores the raw recording data for meetings that have recording markers indefinately. This might include parts of the session where BBB was not supposed to record.
#### Resolution
Delete raw recording data after a meeting has been archived.

### All recordings are always accessible when using Greenlight and BigBlueButton together
#### Description
While greenlight allows unlisting recordings (the default), this does not mean that the recording is not accessible via its direct link.
#### Resolution
Added a third value for the listing setting (public/unlisted/private). Via an additional auth-hook on the recording server, recordings are not displayed when set to private (new default).
Unlisted and Public keep their semantics.

### Cache files

### Retention of cache files

## Logs

### General log rotation
If users opt to keep recording files and logs, they should know and change:

- the duration for which to keep to keep the recording files in /etc/cron.daily/bigluebutton
- the logrotate setttings in /etc/logrotate.d

#### Defaults
By default logs are stored with a retention period of two weeks.

### nginx

### freeswitch

### red5

### kurento

### Meteor/Node.js


# Greenlight

## BigBlueButton always records when recording of a room is enabled
### Description
When used with the default frontend, greenlight, bbb always creates a recording of rooms, even if the recording button is not pressed.

## Greenlight does not request consent to a privacy policy and/or recording of a session when joining a room as a guest.
### Description
BigBlueButton has no feature that forces participants to consent to a privacy policy or being recorded prior to joining a room.
### Resolution
At the moment, I just declare this issue in my privacy policy. For the future, I want to patch GL to request consent before joining.

## Greenlight includes user-names in room urls
In Greenlight, room URLs contain the username of the owner, which might also be private data. Solving this depends on https://github.com/bigbluebutton/greenlight/issues/1057

## Terms
Greenlight supports displaying of terms and conditions for registered users/upon registration. See: http://docs.bigbluebutton.org/greenlight/gl-config.html#adding-terms-and-conditions

## Logs

### Rails Logs
$GREENLIGHTDIR/logs/production.log

### nginx Logs

# coturn

## Logs
