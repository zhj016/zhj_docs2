#Android Volume 调节流程


[android声音调整源代码分析](http://my.oschina.net/u/589963/blog/197732)



PhoneWindow.java
	
	  /**
	1751     * A key was pressed down and not handled by anything else in the window.
	1752     *
	1753     * @see #onKeyUp
	1754     * @see android.view.KeyEvent
	1755     */
	1756    protected boolean onKeyDown(int featureId, int keyCode, KeyEvent event) {
	1757        /* ****************************************************************************
	1758         * HOW TO DECIDE WHERE YOUR KEY HANDLING GOES.
	1759         *
	1760         * If your key handling must happen before the app gets a crack at the event,
	1761         * it goes in PhoneWindowManager.
	1762         *
	1763         * If your key handling should happen in all windows, and does not depend on
	1764         * the state of the current application, other than that the current
	1765         * application can override the behavior by handling the event itself, it
	1766         * should go in PhoneFallbackEventHandler.
	1767         *
	1768         * Only if your handling depends on the window, and the fact that it has
	1769         * a DecorView, should it go here.
	1770         * ****************************************************************************/
	1771
	1772        final KeyEvent.DispatcherState dispatcher =
	1773                mDecor != null ? mDecor.getKeyDispatcherState() : null;
	1774        //Log.i(TAG, "Key down: repeat=" + event.getRepeatCount()
	1775        //        + " flags=0x" + Integer.toHexString(event.getFlags()));
	1776
	1777        switch (keyCode) {
	1778            case KeyEvent.KEYCODE_VOLUME_UP:
	1779            case KeyEvent.KEYCODE_VOLUME_DOWN:
	1780            case KeyEvent.KEYCODE_VOLUME_MUTE: {
	1781                int direction = 0;
	1782                switch (keyCode) {
	1783                    case KeyEvent.KEYCODE_VOLUME_UP:
	1784                        direction = AudioManager.ADJUST_RAISE;
	1785                        break;
	1786                    case KeyEvent.KEYCODE_VOLUME_DOWN:
	1787                        direction = AudioManager.ADJUST_LOWER;
	1788                        break;
	1789                    case KeyEvent.KEYCODE_VOLUME_MUTE:
	1790                        direction = AudioManager.ADJUST_TOGGLE_MUTE;
	1791                        break;
	1792                }
	1793                // If we have a session send it the volume command, otherwise
	1794                // use the suggested stream.
	1795                if (mMediaController != null) {
	1796                    mMediaController.adjustVolume(direction, AudioManager.FLAG_SHOW_UI);
	1797                } else {
	1798                    MediaSessionLegacyHelper.getHelper(getContext()).sendAdjustVolumeBy(
	1799                            mVolumeControlStreamType, direction,
	1800                            AudioManager.FLAG_SHOW_UI | AudioManager.FLAG_VIBRATE
	1801                                    | AudioManager.FLAG_FROM_KEY);
	1802                }
	1803                return true;
	1804            }
	
	
	/frameworks/base/services/core/java/com/android/server/media/MediaSessionService.java
	
	
			
			869        private void dispatchAdjustVolumeLocked(int suggestedStream, int direction, int flags,
		870                MediaSessionRecord session) {
		871            if (DEBUG) {
		872                String description = session == null ? null : session.toString();
		873                Log.d(TAG, "Adjusting session " + description + " by " + direction + ". flags="
		874                        + flags + ", suggestedStream=" + suggestedStream);
		875
		876            }
		877            boolean preferSuggestedStream = false;
		878            if (isValidLocalStreamType(suggestedStream)
		879                    && AudioSystem.isStreamActive(suggestedStream, 0)) {
		880                preferSuggestedStream = true;
		881            }
		882            if (session == null || preferSuggestedStream) {
		883                if ((flags & AudioManager.FLAG_ACTIVE_MEDIA_ONLY) != 0
		884                        && !AudioSystem.isStreamActive(AudioManager.STREAM_MUSIC, 0)) {
		885                    if (DEBUG) {
		886                        Log.d(TAG, "No active session to adjust, skipping media only volume event");
		887                    }
		888                    return;
		889                }
		890                try {
		891                    String packageName = getContext().getOpPackageName();
		892                    mAudioService.adjustSuggestedStreamVolume(direction, suggestedStream,
		893                            flags, packageName, TAG);
		894                } catch (RemoteException e) {
		895                    Log.e(TAG, "Error adjusting default volume.", e);
		896                }
		897            } else {
		898                session.adjustVolume(direction, flags, getContext().getPackageName(),
		899                        UserHandle.myUserId(), true);
		900            }
		901        }
	
	
AudioServie

	
		 private void adjustStreamVolume(int streamType, int direction, int flags,
		1103            String callingPackage, String caller, int uid) {
	
		......
		
		1244            } else if (streamState.adjustIndex(direction * step, device, caller)
		1245                    || streamState.mIsMuted) {

	
	
			  public boolean adjustIndex(int deltaIndex, int device, String caller) {
		3781            return setIndex(getIndex(device) + deltaIndex, device, caller);
		3782        }
		3783
		
setIndex send out broadcast
	
		
		3784        public boolean setIndex(int index, int device, String caller) {
		
		
		3817            if (changed) {
		3818                oldIndex = (oldIndex + 5) / 10;
		3819                index = (index + 5) / 10;
		3820                // log base stream changes to the event log
		3821                if (mStreamVolumeAlias[mStreamType] == mStreamType) {
		3822                    if (caller == null) {
		3823                        Log.w(TAG, "No caller for volume_changed event", new Throwable());
		3824                    }
		3825                    EventLogTags.writeVolumeChanged(mStreamType, oldIndex, index, mIndexMax / 10,
		3826                            caller);
		3827                }
		3828                // fire changed intents for all streams
		3829                mVolumeChanged.putExtra(AudioManager.EXTRA_VOLUME_STREAM_VALUE, index);
		3830                mVolumeChanged.putExtra(AudioManager.EXTRA_PREV_VOLUME_STREAM_VALUE, oldIndex);
		3831                mVolumeChanged.putExtra(AudioManager.EXTRA_VOLUME_STREAM_TYPE_ALIAS,
		3832                        mStreamVolumeAlias[mStreamType]);
		3833                sendBroadcastToAll(mVolumeChanged);
		3834            }
		3835            return changed;
		3836        }


/frameworks/base/packages/SystemUI/src/com/android/systemui/volume/VolumeDialogController.java
	
	if (action.equals(AudioManager.VOLUME_CHANGED_ACTION)) {
	784                final int stream = intent.getIntExtra(AudioManager.EXTRA_VOLUME_STREAM_TYPE, -1);
	785                final int level = intent.getIntExtra(AudioManager.EXTRA_VOLUME_STREAM_VALUE, -1);
	786                final int oldLevel = intent
	787                        .getIntExtra(AudioManager.EXTRA_PREV_VOLUME_STREAM_VALUE, -1);
	788                if (D.BUG) Log.d(TAG, "onReceive VOLUME_CHANGED_ACTION stream=" + stream
	789                        + " level=" + level + " oldLevel=" + oldLevel);
	790                changed = updateStreamLevelW(stream, level);
	791            } 

/frameworks/base/packages/SystemUI/src/com/android/systemui/volume/VolumeDialog.java
	
	875        @Override
	876        public void onStateChanged(State state) {
	877            onStateChangedH(state);
	878        }
	
		
onStateChangedH

		607    private void onStateChangedH(State state) {
	608        final boolean animating = mMotion.isAnimating();
	609        if (D.BUG) Log.d(TAG, "onStateChangedH animating=" + animating);
	610        mState = state;
	611        if (animating) {
	612            mPendingStateChanged = true;
	613            return;
	614        }
	615        mDynamic.clear();
	616        // add any new dynamic rows
	617        for (int i = 0; i < state.states.size(); i++) {
	618            final int stream = state.states.keyAt(i);
	619            final StreamState ss = state.states.valueAt(i);
	620            if (!ss.dynamic) continue;
	621            mDynamic.put(stream, true);
	622            if (findRow(stream) == null) {
	623                addRow(stream, R.drawable.ic_volume_remote, R.drawable.ic_volume_remote_mute, true);
	624            }
	625        }
	626
	627        if (mActiveStream != state.activeStream) {
	628            mActiveStream = state.activeStream;
	629            updateRowsH();
	630            rescheduleTimeoutH();
	631        }
	632        for (VolumeRow row : mRows) {
	633            updateVolumeRowH(row);
	634        }
	635        updateFooterH();
	636    }
	
	
	
/frameworks/base/media/java/android/media/AudioManager.java
		
	742    /**
	743     * @hide
	744     */
	745    public void handleKeyUp(KeyEvent event, int stream) {
	746        int keyCode = event.getKeyCode();
	747        switch (keyCode) {
	748            case KeyEvent.KEYCODE_VOLUME_UP:
	749            case KeyEvent.KEYCODE_VOLUME_DOWN:
	750                /*
	751                 * Play a sound. This is done on key up since we don't want the
	752                 * sound to play when a user holds down volume down to mute.
	753                 */
	754                if (mUseVolumeKeySounds) {
	755                    adjustSuggestedStreamVolume(
	756                            ADJUST_SAME,
	757                            stream,
	758                            FLAG_PLAY_SOUND);
	759                }
	760                mVolumeKeyUpTime = SystemClock.uptimeMillis();
	761                break;
	762            case KeyEvent.KEYCODE_VOLUME_MUTE:
	763                MediaSessionLegacyHelper.getHelper(getContext())
	764                        .sendVolumeKeyEvent(event, false);
	765                break;
	766        }
	767    }