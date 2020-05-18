'''Contributors:'''
- basisbit
- prlanzarin
- zner0L

In many countries there exist regulation of data protection. For operators of
BigBlueButton services, especially private data protection regulations are an
important topic. In the European Union the relevant regulation is the General
Data Protection Regulation (GDPR). Many other countries also adopted similar
regulations (for example the California Consumer Privacy Act (CCPA) in
California, USA) and some of these regulation even have to be complied with
outside of these countries under certain conditions. The following
documentation is supposed to help understand where private data gets processed
and stored in a typical setup of BigBlueButton, and what configuration options
there are.

# BigBlueButton

This section documents privacy related settings, defaults, and configuration
options in BigBlueButton itself. Keep in mind that your configration changes
here may be silently overwritten upon upgrades via apt, see
https://github.com/bigbluebutton/bigbluebutton/issues/9111 To prevent this,
make sure to follow
http://docs.bigbluebutton.org/2.2/customize.html#automatically-apply-configuration-changes-on-restart
to ensure changes are retained upon upgrades and restarts.

## Recordings
### BigBlueButton always records when recording of a room is enabled

When a room is created in BBB that allows recordings (i.e., the recording
button is present) BBB will always record the session.  This is independent of
the button actually being pressed.  The technical reason behind this is that
parts of the recordings (esp. the SVG files for the Whiteboard) depend on
earlier state to be properly processed, see:
https://docs.bigbluebutton.org/dev/recording.html By default these files are
stored for two weeks (see 'Retention of Cache Files' below).  Furthermore,
depending on the use-case and jurisdiction it might be prudent to retain the
option to create 'retroactive' recordings, e.g., when users forgot to click the
recording button.

#### Resolution

Operators have two options for handling this:

##### Globally disable recordings in BBB

Users can change `/usr/share/bbb-web/WEB-INF/classes/bigbluebutton.properties`
and adjust `disableRecordingDefault=false` to `disableRecordingDefault=true` to
globally disable recordings. Furthermore, Recording of breakout rooms should be
disabled by setting `breakoutRoomsRecord=false`. In general, this is an
advisable idea (independent of `disableRecordingDefault=true`, as break-out
rooms might imply a certain level of privacy.

##### Deploy a post-recording script that removes recordings without recording markers 

Users can deploy a custom script that purges recordings and cache-files of
recordings if no recording markers are present. 

Simple version:
`/etc/sudoers:`  
`bigbluebutton ALL = NOPASSWD: /usr/bin/bbb-record`

and

`/usr/local/bigbluebutton/core/scripts/archive/archive.rb (after line 242):`  
`	  BigBlueButton.logger.info("There's no recording marks for #{meeting_id}, deleting the recording.")`  
`	  system('sudo', 'bbb-record', '--delete', "#{meeting_id}") || raise('Failed to delete local recording')`  

For a more complete version that also explicitly deletes cache files of
recordings for freeswitch/kurento, please see:
https://github.com/Kalagon/bbb-recording-archive-workaround 

### BigBlueButton stores presentations froms sessions, even if recordings are disabled

BBB stores the presentations of sessions in /var/bigbluebutton even if the room 
is started with record=false.

#### Resolution

Unknown, unclear which post-recording actions are triggered if no recording is
created.

### BigBlueButton stores full raw recording data

BigBlueButton stores the raw recording data for meetings that have recording
markers indefinately. This might include parts of the session where BBB was not
supposed to record.

#### Resolution

This can be resolved by deleting raw recording data after a meeting has been
archived.  For setups using scalelite, this can be achieved by the following
changes:

`/etc/sudoers:`  
`bigbluebutton ALL = NOPASSWD: /usr/bin/bbb-record`  

`/usr/local/bigbluebutton/core/scripts/post_publish/scalelite_post_publish.rb`, after line 66:  
`system('sudo', 'bbb-record', '--delete', "#{meeting_id}") || raise('Failed to delete local recording')`  

For other systems, removal of `/var/bigbluebutton/recording/raw/$meeting/`
should be added to the post-archive script.

This data can also be removed periodically, see `Retention of cache files`

### All recordings are always accessible

By default, BBB recordings are accessible, see e.g.,
https://github.com/bigbluebutton/bigbluebutton/issues/8505 Additionally, the
URLs for recordings are easily enumerable, see
https://github.com/bigbluebutton/greenlight/issues/1466 and
https://github.com/bigbluebutton/bigbluebutton/issues/9443

#### Resolution

Users can change the nginx configuration to restrict access to the recording
URLs.  Depending on the use-case, the auth statement in nginx can interact with
the used frontend to enforce further restrictions (e.g., requesting the same
credentials as the frontend).  For an example of configuration options and an
authentication callback to a greenlight frontend, see:
https://github.com/ichdasich/bbb-rec-perm

### Cache files

The stack of BBB creates various cache files when recordings are enabled.
Specifically in:

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

For automatically cleaning up these files after recordings, see `BigBlueButton
always records when recording of a room is enabled`.  In addition, to prevent
this data to be written to disk, these files can be mounted with tmpfs to keep
recording caches in-memory.  For this, the following lines have to be added to
`/etc/fstab`:


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

Furthermore, the uid/gid values have to be adjusted to the local installation.
The above example assumes:

`red5:x:999:999:red5 user-daemon:/usr/share/red5:/bin/false`  
`bigbluebutton:x:998:998:bigbluebutton:/home/bigbluebutton:/bin/false`  
`freeswitch:x:997:997:freeswitch:/opt/freeswitch:/bin/bash`  
`kurento:x:996:996::/var/lib/kurento:`  


### Retention of cache files

BBB retains various cache and log files. The general retention period for these files can be configured in `/etc/cron.daily/bigbluebutton`.

- `history` is the retention period for presentations, red5 caches, kurento caches, and freeswitch caches in days
- `unrecorded_days` the retention periods of recordings for meetings which have no recording markers set (user expectation: Were not recorded)
- `publised_days` the retention period of recordings' raw data, if these got published. To enable this, the line `#remove_raw_of_published_recordings` also has to be uncommented

For directly deleting these caches, please see 
'Deploy a post-recording script that removes recordings without recording markers'

### BigBlueButton does not delete recordings when it receives a deleteRecording request

A stand-alone BBB server (unlike with scalelite) does not delete recordings 
when it receives a deleteRecording request via the API. Instead, it moves the
recording to `/var/bigbluebutton/deleted` and unpublishes it.

#### Resolution

There is no easy resolution configurable. Users are advised to create a
high-frequency cron-job, that deletes the contents of
`/var/bigbluebutton/deleted`.

## Logs

### General log rotation

If users opt to keep recording files and logs, they should know and change:

- the duration for which to keep to keep the recording files in `/etc/cron.daily/bigluebutton`, using the `log_history` setting
- the logrotate setttings in `/etc/logrotate.d/(bbb-record-core.logrotate|bbb-webrtc-sfu.logrotate)` which rotate logs in `/var/log/bbb-webrtc-sfu/` every 7 days, and `/var/log/bigbluebutton/(bbb-rap-worker.log|sanity.log)` every 8 days

### BigBlueButton logging
This logs all interactions with rooms, i.e., when they were created, who joined 
when, and when they left in /var/log/bigbluebutton/bbb-web.log

The log verbosity of the core application can be reduced by setting
`appLogLevel=Error` in
`/usr/share/bbb-web/WEB-INF/classes/bigbluebutton.properties`.

### nginx

By default bbb has full access logs enabled for nginx. This includes users IP
addresses, usernames, joined meetings, etc.  This can be disable by switching
the loglevel to 'ERROR' only in `/etc/nginx/sites-available/bigbluebutton` and
`/etc/nginx/nginx.conf`:

`error_log /var/log/nginx/bigbluebutton.error.log;`  
`access_log /dev/null;`  

### freeswitch

Freeswitch, as installed, by default logs with loglevel DEBUG. This can be
changed in `/etc/bbb-fsesl-akka/application.conf` and
`/etc/bbb-fsesl-akka/logback.xml`.  The default configuration Stores usernames,
joined sessions and timestamps.

### red5

Red5 seems to only log meeting ids and timestamps of when video feeds are
started. Still is configured for log-level 'info', which spills the disk. Can
be configured in `/etc/red5/logback.xml`.

### kurento

Logs session names and timestamps, as well as user IP addresses. This also
includes user IP addresses behind NATs, i.e., the actual client addresses,
potentially making users identifiable accross sessions.  Can be configured in
`/etc/default/kurento-media-server`, see
https://doc-kurento.readthedocs.io/en/latest/features/logging.html

Note, that this can most likely be overridden by kurento's systemd unit file.
Hence, `--gst-debug-level=1` should also be set in `/usr/lib/systemd/system/kurento-media-server.service`.

## Integrations

### TURN server

BBB uses a TURN server for nat traversal. By default, a STUN server from
freeswitch.org is configured for BBB. Set up your own TURN/STUN server, and
configure it in
`/usr/share/bbb-web/WEB-INF/classes/spring/turn-stun-servers.xml`, see
https://docs.bigbluebutton.org/2.2/setup-turn-server.html 

### sip.js 

There is a hardcoded google STUN server in
`/usr/share/meteor/bundle/programs/web.browser/app/compatibility/sip.js`.
However, this should never be used, as a failed `GET` of the TURN/STUN settings
in BBB-web should return an empty array, overwriting this setting. Nonetheless,
careful users may want to replace this default.

### Hardcoded google STUN server for kurento

Kurento uses a google STUN server to identify its own external IP address on
each ICE connect. This can be changed in `/etc/kurento/modules/kurento/WebRtcEndpoint.conf.ini`,
by statically configuring the external interface and disabling the stun server 
if the kurento system itself does not sit behind NAT.

### Kurento client side defaults

For client side kurento activities (listen only, screenshare, video) there is a
public fallback STUN server pool which is used if the bbb-web GET fails. There
is work on removing this default.

# Greenlight

## BigBlueButton always records when recording of a room is enabled

When using an upstream greenlight, bbb rooms are always created with
'record=true'. This leads bbb to record full sessions, even if the recording
button is not pressed, see above.

### Resolution

There exists a pull-request that adds per-room enabling of the recording
feature, see: https://github.com/bigbluebutton/greenlight/issues/1163

## Greenlight does not request consent to a privacy policy and/or recording of a session when joining a room as a guest.

BigBlueButton has no feature that forces participants to consent to a privacy
policy or being recorded prior to joining a room.

### Resolution

In https://github.com/bigbluebutton/greenlight/issues/1163 it is discussed how
this can be addressed.
https://github.com/bigbluebutton/greenlight/blob/52ed7150f68c6c66c0488374ccc3457d30fd09d4/app/views/rooms/components/_room_event.html.erb#L21
adds a disclaimer.  The code could be extended to provide a link to the privacy
policy, and a checkbox users have to press to acknowledge that the meeting they
are joining will be recorded, and that they consent to this; The join button
should only be enabled when that checkmark is set.

A more substantial change to BBB and greenlight would be selective recording,
i.e., excluding video, audio, chat and drawing input from users that did not
consent to a recording.

## Greenlight includes user-names in room urls

In Greenlight, room URLs contain the username of the owner, which might also be
private data. Solving this depends on
https://github.com/bigbluebutton/greenlight/issues/1057

## Terms

Greenlight supports displaying of terms and conditions for registered
users/upon registration. See:
http://docs.bigbluebutton.org/greenlight/gl-config.html#adding-terms-and-conditions

## All recordings are always accessible

By default, BBB recordings are accessible, see e.g.,
https://github.com/bigbluebutton/bigbluebutton/issues/8505 Additionally, the
URLs for recordings are easily enumerable, see
https://github.com/bigbluebutton/greenlight/issues/1466 and
https://github.com/bigbluebutton/bigbluebutton/issues/9443 Greenlight, by
default provides two settings for recordings: public, i.e., the recording is
listed on the room page, and unlisted, i.e., the recording is only accessible
via the (enumerable) direct link.

#### Resolution

Greenlight can be patched to allow more fine-grained control of recording
visibility and publications.  With
https://github.com/ichdasich/greenlight/tree/rec_restrictions a third value is
added to the recording visibility pull-down menu (public/unlisted/private),
which leads to corresponding recording metadata being set
(gl-listed=true/unlisted/false) in the recording either in scalelite or on the
bbb host.

https://github.com/ichdasich/bbb-rec-perm can then be used to check requests
for recordings against that metadata, returning a 403 if the recording is set
to private, and returning the recording if it is set to unlisted or public.

## Logs

### Rails Logs

By default, greenlight logs to $GREENLIGHTDIR/logs/production.log. Removing
that line from .env does keep the logs, but does not expose them to the docker
host.  The easiest sollution is linking GREENLIGHTDIR/logs/production.log to
/dev/null

### nginx Logs

By default greenlight has full access logs enabled for nginx. This includes
users IP addresses, usernames, joined meetings, etc.  This can be disable by
switching the loglevel to 'ERROR' only in
`/etc/nginx/sites-available/bigbluebutton` and `/etc/nginx/nginx.conf`:

`error_log /var/log/nginx/bigbluebutton.error.log;`  
`access_log /dev/null;`  

# coturn

By default, coturn logs to /var/log/coturn.log, with regular log-rotation. This
logfile includes IP addresses of users using the TURN server, and the ports
they use. Together with other BBB logs this enables identification of the
sessions these users joined.

coturn has no option of restricting logging. The best option here is linking
`/var/log/coturn.log` to `/dev/null`

# scalelite

## Logs

Scalelite uses a container running nginx as frontend, which accumulates
standard access logs.  By default scalelite has full access logs enabled for
nginx. This includes users IP addresses, usernames, joined meetings, watched
meetings etc.  The easiest way to address this is migrating the nginx to the
host. 

Then, this can be disable by switching the loglevel to 'ERROR' only in
`/etc/nginx/sites-available/bigbluebutton` and `/etc/nginx/nginx.conf`:

`error_log /var/log/nginx/bigbluebutton.error.log;`  
`access_log /dev/null;`  

# General GDPR Considerations

The GDPR lists several important considerations, especially:

- The right to request a (machine readable) aggregate of all private data stored
- The right to have all personal data be erased

Both pose challenges in the context of BBB, especially when considering that
rooms can be joined anonymously/ This means that an operator may not be able to
identify all private information connected to a user easily, even though it is
clearly personal, e.g., webcam recordings of a room joined to as a guest with a
pseudonym. 
