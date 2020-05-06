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

### BigBlueButton stores presentations froms sessions, even if recordings are disabled
BBB stores the presentations of sessions in /var/bigbluebutton even if a recording is disabled. 

#### Resolution
Unknown, unclear which post-recording actions are triggered if no recording is created.

### BigBlueButton stores full raw recording data
BigBlueButton stores the raw recording data for meetings that have recording markers indefinately. This might include parts of the session where BBB was not supposed to record.

#### Resolution
This can be resolved by deleting raw recording data after a meeting has been archived.
For setups using scalelite, this can be achieved by the following changes:

`/etc/sudoers:`
`bigbluebutton ALL = NOPASSWD: /usr/bin/bbb-record`

`/usr/local/bigbluebutton/core/scripts/post_publish/scalelite_post_publish.rb, after line 66:`
`system('sudo', 'bbb-record', '--delete', "#{meeting_id}") || raise('Failed to delete local recording')`

For other systems, removal of `/var/bigbluebutton/recording/raw/$meeting/` should be added to the post-archive script.

This data can also be removed periodically, see `Retention of cache files`

### All recordings are always accessible
By default, BBB recordings are accessible, see e.g., https://github.com/bigbluebutton/bigbluebutton/issues/8505
Additionally, the URLs for recordings are easily enumerable, see https://github.com/bigbluebutton/greenlight/issues/1466 and https://github.com/bigbluebutton/bigbluebutton/issues/9443

#### Resolution
Users can change the nginx configuration to restrict access to the recording URLs.
Depending on the use-case, the auth statement in nginx can interact with the used frontend to enforce further restrictions (e.g., requesting the same credentials as the frontend). 
For an example of configuration options and an authentication callback to a greenlight frontend, see: https://github.com/ichdasich/bbb-rec-perm

### Cache files
The stack of BBB creates various cache files when recordings are enabled. Specifically in:

`/var/bigbluebutton/recording/raw/`
`/var/bigbluebutton/unpublished/`
`/var/bigbluebutton/published/presentation/`
`/usr/share/red5/webapps/video/streams/`
`/usr/share/red5/webapps/screenshare/streams/`
`/usr/share/red5/webapps/video-broadcast/streams/`
`/var/kurento/recordings/`
`/var/kurento/screenshare/`
`/var/freeswitch/meetings/`

#### Resolution
For automatically cleaning up these files after recordings, see `BigBlueButton always records when recording of a room is enabled`.
In addition, to prevent this data to be written to disk, these files can be mounted with tmpfs to keep recording caches in-memory. 
For this, the following lines have to be added to `/etc/fstab`:


`tmpfs /var/bigbluebutton/recording/raw/ tmpfs rw,nosuid,noatime,uid=998,gid=998,size=16G,mode=0755   0    0`
`tmpfs /var/bigbluebutton/unpublished/ tmpfs rw,nosuid,noatime,uid=998,gid=998,size=16G,mode=0755   0    0`
`tmpfs /var/bigbluebutton/published/presentation/ tmpfs rw,nosuid,noatime,uid=998,gid=998,size=16G,mode=0755   0    0`
`tmpfs /usr/share/red5/webapps/video/streams/ tmpfs rw,nosuid,noatime,uid=999,gid=999,size=16G,mode=0755   0    0`
`tmpfs /usr/share/red5/webapps/screenshare/streams/ tmpfs rw,nosuid,noatime,uid=999,gid=999,size=16G,mode=0755   0    0`
`tmpfs /usr/share/red5/webapps/video-broadcast/streams/ tmpfs rw,nosuid,noatime,uid=999,gid=999,size=16G,mode=0755   0    0`
`tmpfs /var/kurento/recordings/ tmpfs rw,nosuid,noatime,uid=996,gid=996,size=16G,mode=0755   0    0`
`tmpfs /var/kurento/screenshare/ tmpfs rw,nosuid,noatime,uid=996,gid=996,size=16G,mode=0755   0    0`
`tmpfs /var/freeswitch/meetings/ tmpfs rw,nosuid,noatime,uid=997,gid=997,size=16G,mode=0755   0    0`

After that, the operator has to execute `bbb-conf --stop; mount -a; bbb-conf --start`.

Furthermore, the uid/gid values have to be adjusted to the local installation. The above example assumes:
`red5:x:999:999:red5 user-daemon:/usr/share/red5:/bin/false`
`bigbluebutton:x:998:998:bigbluebutton:/home/bigbluebutton:/bin/false`
`freeswitch:x:997:997:freeswitch:/opt/freeswitch:/bin/bash`
`kurento:x:996:996::/var/lib/kurento:`


### Retention of cache files
BBB retains various cache and log files. The general retention period for these files can be configured in `/etc/cron.daily/bigbluebutton`.

- `history` is the retention period for presentations, red5 caches, kurento caches, and freeswitch caches in days
- `unrecorded_days` the retention periods of recordings for meetings which have no recording markers set (user expectation: Were not recorded)
- `publised_days` the retention period of recordings' raw data, if these got published. To enable this, the line `#remove_raw_of_published_recordings` also has to be uncommented

For directly deleting these caches, please see `Deploy a post-recording script that removes recordings without recording markers`

## Logs

### General log rotation
If users opt to keep recording files and logs, they should know and change:

- the duration for which to keep to keep the recording files in `/etc/cron.daily/bigluebutton`, using the `log_history` setting
- the logrotate setttings in `/etc/logrotate.d/(bbb-record-core.logrotate|bbb-webrtc-sfu.logrotate)` which rotate logs in `/var/log/bbb-webrtc-sfu/` every 7 days, and `/var/log/bigbluebutton/(bbb-rap-worker.log|sanity.log)` every 8 days

### nginx
By default bbb has full access logs enabled for nginx. This includes users IP addresses, usernames, joined meetings, etc. 
This can be disable by switching the loglevel to 'ERROR' only in `/etc/nginx/sites-available/bigbluebutton` and `/etc/nginx/nginx.conf`:
`error_log /var/log/nginx/bigbluebutton.error.log;`
`access_log /dev/null;`

### freeswitch
Freeswitch, as installed, by default logs with loglevel DEBUG. This can be changed in `/etc/bbb-fsesl-akka/application.conf` and `/etc/bbb-fsesl-akka/logback.xml`.
The default configuration Stores usernames, joined sessions and timestamps.

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
