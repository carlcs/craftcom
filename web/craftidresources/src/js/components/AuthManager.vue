<template>
    <div>
        <!-- logout warning modal -->
        <modal :show.sync="showingLogoutWarningModal" :transition.sync="logoutWarningModalTransitionName" @after-enter="onAfterEnterLogoutWarningModal" class="auth-manager-modal">
            <template slot="body">
                <i class="fa fa-warning"></i>
                {{ logoutWarningPara }}

                <div class="float-right mt-5">
                    <a @click.prevent="logout" href="#" class="btn btn-secondary">Logout now</a>
                    <a @click.prevent="renewSession" ref="renewSessionBtn" href="#" class="btn btn-primary">Keep me logged in</a>
                </div>
            </template>
        </modal>

        <!-- login modal -->
        <modal :show.sync="showingLoginModal" :transition.sync="loginModalTransitionName" @after-enter="onAfterEnterLoginModal" @leave="onLeaveLoginModal" class="auth-manager-modal">
            <template slot="body">
                <i class="fa fa-warning"></i>
                <form @submit.prevent="login">
                    <h6>Your session has ended</h6>
                    <p>Enter your password to log back in.</p>

                    <div>
                        <div class="d-flex">
                            <input v-model="password" ref="passwordInput" placeholder="Password" type="password" id="password" class="form-control mr-2"  :class="{'is-invalid': loginErrorPara }" />
                            <input type="submit" class="btn btn-primary mr-2" value="Login" :disabled="!passwordValidates" />
                            <div class="spinner" :class="{'invisible': !passwordSpinner}"></div>
                        </div>
                    </div>

                    <div class="text-danger" v-if="loginErrorPara">{{ loginErrorPara }}</div>
                </form>
            </template>
        </modal>
    </div>
</template>


<script>
    import axios from 'axios';
    import qs from 'qs';
    import Modal from './Modal';
    import IsMobileBrowser from './IsMobileBrowser';
    import humanizeDuration from 'humanize-duration';

    export default {

        components: {
            Modal,
        },

        mixins: [IsMobileBrowser],

        data() {
            return {
                showingLoginModal: false,
                showingLogoutWarningModal: false,
                logoutWarningPara: null,
                loginErrorPara: null,
                password: null,
                passwordValidates: false,
                logoutWarningModalTransitionName: 'fade',
                loginModalTransitionName: 'quick-fade',
                passwordSpinner: false,

                minSafeSessionTime: 120,
                checkInterval: 60,
            }
        },

        methods: {

            /**
             * Sets a timer for the next time to check the auth timeout.
             */
            setCheckRemainingSessionTimer(seconds) {
                if (this.checkRemainingSessionTimer) {
                    clearTimeout(this.checkRemainingSessionTimer);
                }

                this.checkRemainingSessionTimer = setTimeout(function() {
                    this.checkRemainingSessionTime();
                }.bind(this), seconds * 1000);
            },

            /**
             * Pings the server to see how many seconds are left on the current user session, and handles the response.
             */
            checkRemainingSessionTime(extendSession) {
                let config = {};

                if(!extendSession) {
                    config.params = {
                        dontExtendSession: 1
                    };
                }

                axios.get(Craft.actionUrl+'/users/get-remaining-session-time', config)
                    .then(response => {
                        if (typeof response.data.csrfTokenValue !== 'undefined' && typeof Craft.csrfTokenValue !== 'undefined') {
                            Craft.csrfTokenValue = response.data.csrfTokenValue;
                        }

                        this.updateRemainingSessionTime(response.data.timeout);
                        this.submitLoginIfLoggedOut = false;
                    })
                    .catch(response => {

                        this.updateRemainingSessionTime(-1);
                    });
            },

            /**
             * Updates our record of the auth timeout, and handles it.
             */
            updateRemainingSessionTime(remainingSessionTime) {
                this.remainingSessionTime = parseInt(remainingSessionTime);

                // Are we within the warning window?
                if (this.remainingSessionTime !== -1 && this.remainingSessionTime < this.minSafeSessionTime) {
                    // Is there still time to renew the session?
                    if (this.remainingSessionTime) {
                        if (!this.showingLogoutWarningModal) {
                            // Show the warning modal
                            this.showLogoutWarningModal();
                        }

                        // Will the session expire before the next checkup?
                        if (this.remainingSessionTime < this.checkInterval) {
                            if (this.showLoginModalTimer) {
                                clearTimeout(this.showLoginModalTimer);
                            }

                            this.showLoginModalTimer = setTimeout(function() {
                                this.showLoginModal();
                            }.bind(this), this.remainingSessionTime * 1000);
                        }
                    } else {
                        if (this.showingLoginModal) {
                            if (this.submitLoginIfLoggedOut) {
                                this.submitLogin();
                            }
                        } else {
                            // Show the login modal
                            this.showLoginModal();
                        }
                    }

                    this.setCheckRemainingSessionTimer(this.checkInterval);
                } else {
                    // Everything's good!
                    this.hideLogoutWarningModal();
                    this.hideLoginModal();

                    // Will be be within the minSafeSessionTime before the next update?
                    if (this.remainingSessionTime !== -1 && this.remainingSessionTime < (this.minSafeSessionTime + this.checkInterval)) {
                        this.setCheckRemainingSessionTimer(this.remainingSessionTime - this.minSafeSessionTime + 1);
                    } else {
                        this.setCheckRemainingSessionTimer(this.checkInterval);
                    }
                }
            },

            /**
             * Shows the logout warning modal.
             */
            showLogoutWarningModal() {
                let quickShow;

                if (this.showingLoginModal) {
                    this.hideLoginModal(true);
                    quickShow = true;
                } else {
                    quickShow = false;
                }

                if (quickShow) {
                    this.logoutWarningModalTransitionName = 'quick-fade';
                } else {
                    this.logoutWarningModalTransitionName = 'fade';
                }

                this.$nextTick(() => {
                    this.showingLogoutWarningModal = true;
                });

                this.updateLogoutWarningMessage();

                this.decrementLogoutWarningInterval = setInterval(function() {
                    this.decrementLogoutWarning()
                }.bind(this), 1000);

            },

            /**
             * On after enter logout warning modal.
             */
            onAfterEnterLogoutWarningModal() {
                if(!this.isMobileBrowser(true)) {
                    this.$refs.renewSessionBtn.focus();
                }
            },

            /**
             * Updates the logout warning message indicating that the session is about to expire.
             */
            updateLogoutWarningMessage() {
                let time = humanizeDuration(this.remainingSessionTime * 1000);
                this.logoutWarningPara = 'Your session will expire in ' + time + '.';
            },

            /**
             * Decrement logout warning.
             */
            decrementLogoutWarning() {
                if (this.remainingSessionTime > 0) {
                    this.remainingSessionTime--;
                    this.updateLogoutWarningMessage();
                }

                if (this.remainingSessionTime === 0) {
                    clearInterval(this.decrementLogoutWarningInterval);
                }
            },

            /**
             * Hides the logout warning modal.
             */
            hideLogoutWarningModal(quick) {
                if(quick) {
                    this.logoutWarningModalTransitionName = 'quick-fade';
                } else {
                    this.logoutWarningModalTransitionName = 'fade';
                }

                this.$nextTick(() => {
                    this.showingLogoutWarningModal = false;
                });

                if (this.decrementLogoutWarningInterval) {
                    clearInterval(this.decrementLogoutWarningInterval);
                }
            },

            /**
             * Shows the login modal.
             */
            showLoginModal() {
                let quickShow;

                if (this.showingLogoutWarningModal) {
                    this.hideLogoutWarningModal(true);
                    quickShow = true;
                } else {
                    quickShow = false;
                }

                if (quickShow) {
                    this.loginModalTransitionName = 'quick-fade';
                } else {
                    this.loginModalTransitionName = 'fade';
                }

                this.$nextTick(() => {
                    this.showingLoginModal = true;
                });
            },

            /**
             * On after enter login modal.
             */
            onAfterEnterLoginModal() {
                if(!this.isMobileBrowser(true)) {
                    this.$refs.passwordInput.focus();
                }
            },

            /**
             * Hides the login modal.
             */
            hideLoginModal(quick) {
                if (quick) {
                    this.loginModalTransitionName = 'quick-fade';
                } else {
                    this.loginModalTransitionName = 'fade';
                }

                this.$nextTick(() => {
                    this.showingLoginModal = false;
                });
            },

            /**
             * On leave login modal.
             */
            onLeaveLoginModal() {
                this.password = '';
            },

            /**
             * Logout.
             */
            logout() {
                axios.get(Craft.actionUrl + '/users/logout')
                    .then(response => {
                        document.location.href = '';
                    })
            },

            /**
             * Renew session.
             */
            renewSession() {
                this.hideLogoutWarningModal();
                this.checkRemainingSessionTime(true);
            },

            /**
             * Validate password.
             */
            validatePassword() {
                if (this.password.length >= 6) {
                    this.passwordValidates = true;
                    return true;
                } else {
                    this.passwordValidates = false;
                    return false;
                }
            },

            /**
             * Login.
             */
            login() {
                if (this.validatePassword()) {
                    this.passwordSpinner = true;

                    this.clearLoginError();

                    if (typeof Craft.csrfTokenValue !== 'undefined') {
                        // Check the auth status one last time before sending this off,
                        // in case the user has already logged back in from another window/tab
                        this.submitLoginIfLoggedOut = true;
                        this.checkRemainingSessionTime();
                    } else {
                        this.submitLogin();
                    }
                }
            },

            /**
             * Submit login.
             */
            submitLogin() {
                let data = {
                    loginName: Craft.username,
                    password: this.password
                };

                let params = qs.stringify(data);
                let headers = {};

                if (Craft.csrfTokenValue && Craft.csrfTokenName) {
                    headers['X-CSRF-Token'] = Craft.csrfTokenValue;
                }

                axios.post(Craft.actionUrl+'/users/login', params, { headers: headers })
                    .then(response => {
                        this.passwordSpinner = false;

                        if (response.data.success) {
                            this.hideLoginModal();
                            this.checkRemainingSessionTime();
                        } else {
                            this.showLoginError(response.data.error);

                            if(!this.isMobileBrowser(true)) {
                                this.$refs.passwordInput.focus();
                            }
                        }
                    })
                    .catch(response => {
                        this.passwordSpinner = false;
                        this.showLoginError();
                    });
            },

            /**
             * Show login error.
             */
            showLoginError(error) {
                if (error === null || typeof error === 'undefined') {
                    error = 'An unknown error occurred.';
                }

                this.loginErrorPara = error;
            },

            /**
             * Clear login error.
             */
            clearLoginError() {
                this.showLoginError('');
            },
        },

        watch: {

            /**
             * Validate password when the password value changes.
             */
            password(newVal) {
                this.validatePassword();

                return newVal;
            }

        },

        mounted() {
            // Let's get it started.
            this.updateRemainingSessionTime(Craft.remainingSessionTime);
        }

    }
</script>
