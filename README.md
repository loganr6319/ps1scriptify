# ps1scriptify

Python script that creates a Powershell function used for calling a Python script.
Easy as pie to understand.

CURRENTLY ONLY WORKS ON WINDOWS AND WITH PYTHON3.

usage:
>python ps1scriptify.py [Python fil#!/usr/bin/python
#
# Copyright Duo Security (C) 2016
#
# This script installs a device certificate for the currently logged in user
# Supports macOS 10.11 and above.
# v3.5

from OpenSSL import crypto
from urlparse import urlparse
from collections import OrderedDict
from SystemConfiguration import SCDynamicStoreCopyConsoleUser
from pwd import getpwnam
from datetime import datetime
from datetime import timedelta

import argparse
import subprocess
import sys
import os
import base64
import hmac
import hashlib
import json
import urllib2
import tempfile
import random
import string


CONFIG = r'{"RENEWAL_WINDOW_DAYS": "MTQ=", "VALID_DAYS": "MzY1", "PASSWORD": "WWU1IWlHMkw9bUhjKyUodnlwclp2TFMz", "AKEY": "REFUNk1BUDlSUE8yRFExWjFGM1A=", "APPLICATION_SECRET": "TGVCWW5qVk1OOW1qUm5QWWFhU29wQ1NJc1ZEaHo2bUR2N09PNDlwMVl4c3dSRHI1ODlsbGRJZHRRRA==", "USERNAME": "RFVPUEtJXGE4eGhwemo2eWhkcWdwNGQ=", "TEMPLATE_NAME": "RHVvRGV2aWNlQXV0aGVudGljYXRpb24=", "APPLICATION_KEY_TEXT": "QjVPVzBOU2dPZ0V3V3VXRWhyMTJ6TkJnYjMzbWw=", "ENROLLMENT_URL": "aHR0cHM6Ly9kdW8xMDFjbXMuY3NzcGtpLmNvbS9DTVNOZXdBUEkvQ2VydEVucm9sbC8zL1BrY3MxMA==", "MKEY": "RE1MS1owVE8xSUpVWUdaTkJJTjQ="}'

ACL_TEAMIDS = [
    'EQHXZ8M8AV',  # Chrome
    'BQR82RBBHL',  # Slack
    'BJ4HAAB9B3',  # Zoom
    'HE4P42JBGN',  # BlueJeans
    'SDLYW7A8F3',  # HipChat
    '3Y9EC8WH48',  # GoToMeeting
    'Q79WDW8YH9',  # Evernote
    '2BUA8C4S2C',  # 1Password
    '57P38MF5GS',  # F5 Big Edge
    'VEKTX9H2N7',  # Electron
    'FMRT79K8DG',  # quip
    'JQ525L2MZD',  # Adobe Air
    '4XF3XNRN6Y',  # Vivaldi
    '477EAT77S3',  # Yandex
    'UBF8T346G9',  # Microsoft
    '483DWKW443',  # JAMF
    'PXPZ95SK77',  # Palo Alto Networks
    'M683GB7CPW',  # Box
    'UL7F34E7DN',  # Opera
    'FNN8Z5JMFP',  # Duo
    'DE8Y96K9QP',  # Cisco
    'BUX8SV4LQA',  # Symphony
]

# Module level constants
CERTNAME = 'Duo Device Authentication'
ROOT_CERTNAME = 'Duo Endpoint Validation Root CA'
INTERMEDIATE_CERTNAME = 'Duo Endpoint Validation Issuing CA'
KEYSIZE = 2048

# We use a specific named file on disk for the key so that we can import
# it to keychain with a specific name
KEYFILE_NAME = 'Duo Device Authentication.pem'
DUO_KEYCHAIN_NAME = 'duo-auth.keychain'
DUO_KEYCHAIN_PASSWORD_LABEL = 'Duo Security'
TEMP_KEYCHAIN_NAME = 'duo-temp.keychain'
TEMP_KEYCHAIN_PASSWORD = 'temp-Password'

LAUNCHAGENT_LABEL = 'com.duosecurity.keychainhelper'
LAUNCHAGENT_FILENAME = LAUNCHAGENT_LABEL + '.plist'
LAUNCHAGENT_TEMPLATE = (
"""<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple Computer//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
   <key>Label</key>
   <string>{label}</string>
   <key>ProgramArguments</key>
    <array>
        <string>bash</string>
        <string>-c</string>
        <string>security unlock-keychain -p $(security find-generic-password -l "Duo Security" -w {login_keychain}) {duo_keychain}</string>
    </array>
   <key>RunAtLoad</key>
   <true/>
   <key>Nice</key>
   <integer>-20</integer>
   <key>ProcessType</key>
   <string>Interactive</string>
</dict>
</plist>""")

# Set in init_consts()
CONSOLEUSER = None
CONSOLEUID = None
HOME_DIR = None
KEYCHAIN = None
LOGIN_KEYCHAIN = None
TEMP_KEYCHAIN = None
HARDWARE_UUID = None
COMPUTER_NAME = None
RUNNING_IN_DELETE_MODE = False
RUNNING_IN_CERT_CHECK_MODE = False

def subprocess_call(command):
    """ Wrapper around subprocess.call that mutes stdout"""
    with open(os.devnull, 'wb') as nullfile:
        return subprocess.call(command, stdout=nullfile)


def as_user(args):
    """
    Wrap the command line args so that the command runs in the
    proper user context
    """
    return ([
        'launchctl', 'asuser', CONSOLEUID,
        'sudo', '-u', CONSOLEUSER,
    ] + args)


def find_certificate_pem():
    return subprocess.check_output(
        as_user([
            'security', 'find-certificate',
            '-a', '-c', CERTNAME,
            '-Z', '-p', KEYCHAIN,
        ])
    )


def get_cert_valid_days():
    certpem = find_certificate_pem()
    cert = crypto.load_certificate(crypto.FILETYPE_PEM, certpem)
    valid_time = (datetime.strptime(cert.get_notAfter(), '%Y%m%d%H%M%SZ') -
                  datetime.strptime(cert.get_notBefore(), '%Y%m%d%H%M%SZ'))
    return valid_time.days


def get_cert_expiration():
    certpem = find_certificate_pem()
    cert = crypto.load_certificate(crypto.FILETYPE_PEM, certpem)
    return datetime.strptime(cert.get_notAfter(), '%Y%m%d%H%M%SZ')


def should_renew_certificate():
    """
    Provided that the leaf cert exists, check to see if the leaf cert
    is within the renew period
    """
    return get_cert_expiration() - datetime.utcnow() < RENEWAL_WINDOW


def get_home_directory():
    """
    Find the home directory for the current logged in user.
    """
    try:
        res = subprocess.check_output(
            as_user(['printenv', 'HOME']))
        return res.strip()
    except Exception:
        return None


def get_uuid():
    """
    Find the hardware UUID of the device using system_profiler
    """
    res = subprocess.check_output([
        'system_profiler', 'SPHardwareDataType'])
    for line in res.splitlines():
        if 'Hardware UUID:' in line:
            return line.replace('Hardware UUID:', '').strip()
    return None


def get_computer_name():
    """
    Find the computer name using scutil
    """
    res = subprocess.check_output([
        'scutil', '--get', 'ComputerName'])
    return res.strip() or None


def cert_has_correct_validity_duration():
    """
    Check if the leaf cert has a valid duration with the correct number
    of days
    """
    valid_days = get_cert_valid_days()
    return (
        valid_days == CERT_VALID_DAYS or

        # Special case for leap years
        (valid_days == 366 and CERT_VALID_DAYS == 365)
    )


def cert_has_required_attributes():
    """
    Check if the leaf cert has the required akey/mkey attributes
    """
    certpem = find_certificate_pem()
    cert = crypto.load_certificate(crypto.FILETYPE_PEM, certpem)
    extension_count = cert.get_extension_count()

    asn_utf8_identifier = '\x0c'
    asn_key_length = '\x14'
    ext_prefix = asn_utf8_identifier + asn_key_length
    asn_akey = ext_prefix + AKEY
    asn_mkey = ext_prefix + MKEY

    has_valid_akey = False
    has_valid_mkey = False
    for i in range(extension_count):
        ext_data = cert.get_extension(i).get_data()
        if ext_data == asn_akey:
            has_valid_akey = True
        if ext_data == asn_mkey:
            has_valid_mkey = True

    return has_valid_mkey and has_valid_akey


def exit_if_running_as_root():
    """
    The script must be ran as the actual user, using root privileges, otherwise
    we will fail to write to the user's keychain.
    """
    if get_home_directory() == '/var/root' or CONSOLEUSER == 'root':
        print('=== Error: This script can not be ran as the root user. '
              'It must be ran as the user with root privileges (sudo).')
        exit(1)


def fetch_config(CONFIG):
    parser = argparse.ArgumentParser(description='This script installs a device certificate for the currently logged in user')

    required_args = ['enrollment_url', 'application_key_text', 'akey',
                        'mkey', 'renewal_window_days', 'template_name',
                        'valid_days', 'username', 'password', 'application_secret']

    for arg in required_args:
        parser.add_argument('--' + arg)

    # some management systems pass in default arguments, only parse known args
    args, _ = parser.parse_known_args()
    has_all_required_args = all(getattr(args, arg) is not None for arg in required_args)
    has_some_required_args = any(getattr(args, arg) is not None for arg in required_args)

    # if any command line args are passed, check that all necessary args are passed
    if has_some_required_args and not has_all_required_args:
        raise Exception("=== All command line args must be passed")

    # if all necessary args are passed use those
    if has_all_required_args:
        return {
            'ENROLLMENT_URL': str(base64.b64decode(args.enrollment_url)),
            'USERNAME': base64.b64decode(args.username).encode('utf-8'),
            'PASSWORD': base64.b64decode(args.password).encode('utf-8'),
            'APPLICATION_KEY': str(args.application_key_text),
            'APPLICATION_SECRET': str(base64.b64decode(args.application_secret)),
            'AKEY': str(base64.b64decode(args.akey)),
            'MKEY': str(base64.b64decode(args.mkey)),
            'RENEWAL_WINDOW': timedelta(days=int(base64.b64decode(args.renewal_window_days))),
            'TEMPLATE_NAME': base64.b64decode(args.template_name),
            'CERT_VALID_DAYS': int(base64.b64decode(args.valid_days))
        }

    # if nothing is set raise
    if CONFIG is '':
        raise Exception("=== No Configuration provided")

    # otherwise use CONFIG
    CONFIG = json.loads(CONFIG)

    return {
        'ENROLLMENT_URL': str(base64.b64decode(CONFIG.get('ENROLLMENT_URL'))),
        'USERNAME': base64.b64decode(CONFIG.get('USERNAME', '')).encode('utf-8'),
        'PASSWORD': base64.b64decode(CONFIG.get('PASSWORD', '')).encode('utf-8'),
        'APPLICATION_KEY': str(CONFIG.get('APPLICATION_KEY_TEXT')),
        'APPLICATION_SECRET': str(base64.b64decode(CONFIG.get('APPLICATION_SECRET', ''))),
        'AKEY': str(base64.b64decode(CONFIG.get('AKEY'))),
        'MKEY': str(base64.b64decode(CONFIG.get('MKEY'))),
        'RENEWAL_WINDOW': timedelta(days=int(base64.b64decode(CONFIG.get('RENEWAL_WINDOW_DAYS')))),
        'TEMPLATE_NAME': base64.b64decode(CONFIG.get('TEMPLATE_NAME')),
        'CERT_VALID_DAYS': int(base64.b64decode(CONFIG.get('VALID_DAYS')))
    }


try:
    config = fetch_config(CONFIG)
    (ENROLLMENT_URL, USERNAME, PASSWORD,
        APPLICATION_KEY, APPLICATION_SECRET,
        AKEY, MKEY, RENEWAL_WINDOW,
        TEMPLATE_NAME, CERT_VALID_DAYS
        ) = (config[k] for k in (
            'ENROLLMENT_URL', 'USERNAME', 'PASSWORD',
            'APPLICATION_KEY', 'APPLICATION_SECRET',
            'AKEY', 'MKEY', 'RENEWAL_WINDOW',
            'TEMPLATE_NAME', 'CERT_VALID_DAYS'))
except Exception as e:
    print('== Error fetching config variables... Error: %s' % e)
    sys.exit(1)


def init_consts():
    """
    Set up additional consts that the module will use
    """
    global CONSOLEUSER
    CONSOLEUSER = str(SCDynamicStoreCopyConsoleUser( None, None, None )[0])

    global CONSOLEUID
    CONSOLEUID = str(getpwnam(CONSOLEUSER).pw_uid)

    global HOME_DIR
    global KEYCHAIN
    global LOGIN_KEYCHAIN
    global TEMP_KEYCHAIN
    HOME_DIR = get_home_directory()
    if not HOME_DIR:
        HOME_DIR = '/Users/%s' % CONSOLEUSER
    LOGIN_KEYCHAIN = os.path.join(HOME_DIR, 'Library', 'Keychains', 'login.keychain')
    KEYCHAIN = os.path.join(HOME_DIR, 'Library', 'Keychains', DUO_KEYCHAIN_NAME)
    TEMP_KEYCHAIN  = os.path.join(HOME_DIR, 'Library', 'Keychains', TEMP_KEYCHAIN_NAME)

    global HARDWARE_UUID
    HARDWARE_UUID = get_uuid()

    global COMPUTER_NAME
    COMPUTER_NAME = get_computer_name()

    # Verify all config arguments are present
    if not all([ENROLLMENT_URL, USERNAME, PASSWORD,
            APPLICATION_KEY, APPLICATION_SECRET]):
        raise Exception('=== Invalid configuration parameters')

    global RUNNING_IN_DELETE_MODE
    global RUNNING_IN_CERT_CHECK_MODE
    for arg in sys.argv:
        if arg == 'DUO-CERTIFICATE-DELETE-MODE':
            RUNNING_IN_DELETE_MODE = True
        if arg == 'DUO-CERTIFICATE-CHECK-MODE':
            RUNNING_IN_CERT_CHECK_MODE = True

    if RUNNING_IN_DELETE_MODE and RUNNING_IN_CERT_CHECK_MODE:
        raise Exception("=== Can not run in both DELETE and CERT CHECK mode!")


def get_hashes(common_name, keychain, require_private_key=False,
               error_log_only=False):
    """
    Get a list of certificate hashes for certificates that match the
    provided CN

    When require_private_key is True, only return valid identities with a
    corresponding private key. Identities without a private keys will be excluded.
    When error_log_only is set to True, only errors are printed.
    """
    if not error_log_only:
        print('=== Finding certificates matching CN=%s' % common_name)

    hashes = []
    if require_private_key:
        # Check identities (public + private key combo)
        res = subprocess.check_output(
            as_user([
                'security', 'find-identity', keychain,
            ])).splitlines()
        # Extract the hash of the identity. Format of the output is:
        # '1) 1B4AC44E7AD6B30341DAFFA7050C6C00BC4303CC "Duo Device Authentication"'
        hash_start_mark = ') '
        for line in res:
            if 'Duo Device Authentication' in line:
                start = line.find(hash_start_mark) + len(hash_start_mark)
                cert_hash = line[start:start+40]
                if cert_hash not in hashes:
                    hashes.append(cert_hash)
    else:
        # Check cert only
        res = subprocess.check_output(
            as_user([
                'security', 'find-certificate',
                '-a', '-c', common_name,
                '-Z', keychain,
            ])).splitlines()

        for line in res:
            # Grab the SHA-1 hashes of the certs that match the CN
            if line.startswith('SHA-1 hash: '):
                hashes.append(line.replace('SHA-1 hash: ', ''))

    if not error_log_only:
        if len(hashes) > 1:
            print('=== Multiple certificates matching CN=%s' % common_name)
        elif len(hashes) == 1:
            print('=== Cert hit for CN=%s' % common_name)
        else:
            print('=== No cert found for CN=%s' % common_name)
    return hashes


def delete_certs(common_name, keychain, error_log_only=False):
    """
    Delete any certificates in the user's keychain that match the provided
    Common Name

    When error_log_only is set to True, only errors are printed.
    """
    if not error_log_only:
        print('=== Deleting identity with CN=%s...' % common_name)
    hashes = get_hashes(common_name, keychain, error_log_only=error_log_only)
    success = True
    for cert_hash in hashes:
        status = subprocess.call(
            as_user([
                'security', 'delete-identity',
                '-Z', cert_hash, keychain
            ]))
        if status != 0:
            # Older Macs don't support delete-identity.
            # Fallback to delete-certificate.
            print('=== Deleting certificate with CN=%s...' % common_name)
            status = subprocess.call(
                as_user([
                    'security', 'delete-certificate',
                    '-Z', cert_hash, keychain
                ]))
        if status != 0:
            success = False
            print('=== Deleting cert failed with status code: %s. common_name: %s, keychain: %s' \
                %(status, common_name, keychain))
    return success


def duo_keychain_exists():
    """
    Check if the custom Duo keychain exists.
    """
    res = subprocess.check_output(
        as_user(['security', 'list-keychains', '-d', 'user']))
    in_search_list = KEYCHAIN in res

    keychain_dir = os.path.join(HOME_DIR, 'Library', 'Keychains')
    file_on_disk = any([
        DUO_KEYCHAIN_NAME in f and os.path.isfile(
            os.path.join(keychain_dir, f)) for f in os.listdir(keychain_dir)])

    return file_on_disk and in_search_list


def get_duo_keychain_password():
    """
    Fetch the Duo keychain password from the user's login keychain.
    """
    try:
        res = subprocess.check_output(
            as_user([
                'security', 'find-generic-password',
                '-l', DUO_KEYCHAIN_PASSWORD_LABEL,
                '-w', LOGIN_KEYCHAIN,
            ]))
        return res.strip() or None
    except Exception:
        return None


def generate_duo_keychain_password():
    """
    Generate a random, 20 character password for the duo keychain
    in the user's login keychain. Returns the password generated.
    """
    print('=== Generating a password for the duo-auth keychain...')
    password = ''.join(random.SystemRandom().choice(
        string.ascii_letters + string.digits) for _ in range(20))
    status = subprocess.call(
        as_user([
            'security', 'add-generic-password',
            '-a', DUO_KEYCHAIN_NAME,
            '-s', DUO_KEYCHAIN_PASSWORD_LABEL,
            '-l', DUO_KEYCHAIN_PASSWORD_LABEL,
            '-w', password,
            '-A', '-U',
            LOGIN_KEYCHAIN,
        ]))
    if status != 0:
        print ('=== Failed to add new password to keychain with status code: %s' %status)
        return None

    return password


def create_duo_keychain(password):
    """
    Create the Duo custom keychain and set its settings so that
    it does not auto-lock. Add this keychain to the search list.
    """
    print('=== Creating the duo-auth keychain...')
    status = subprocess.call(
        as_user([
            'security', 'create-keychain',
            '-p', password, KEYCHAIN,
        ]))
    if status != 0:
        print ('=== Create keychain failed with status: %s' %status)
        return False

    status = subprocess.call(
        as_user(['security', 'set-keychain-settings', KEYCHAIN]))
    if status != 0:
        print ('=== Setting keychain settings failed with status: %s' %status)
        return False

    status = subprocess.call(
        as_user([
            'security', 'list-keychains',
            '-d', 'user',
            '-s', LOGIN_KEYCHAIN, KEYCHAIN,
        ]))
    if status != 0:
        print ('=== Listing keychain failed with status: %s' %status)

    return status == 0


def unlock_duo_keychain(password):
    """
    Unlock the Duo keychain with the password provided.
    """
    print('=== Unlocking the duo-auth keychain...')
    status = subprocess.call(
        as_user([
            'security', 'unlock-keychain',
            '-p', password, DUO_KEYCHAIN_NAME,
        ]))
    if status != 0:
        print('=== Unlocking duo-auth keychain failed with status code: %s.' %status)
    return status == 0


def lock_duo_keychain():
    """
    Lock the Duo keychain.
    """
    subprocess.call(
        as_user([
            'security', 'lock-keychain',
            DUO_KEYCHAIN_NAME,
        ]))


def delete_duo_keychain():
    """
    Delete the Duo keychain.
    """
    print('=== Deleting the duo-auth keychain...')
    status = subprocess.call(
        as_user(['security', 'delete-keychain', KEYCHAIN]))

    if status != 0:
        print('=== Deleting duo-auth keychain failed with status code: %s.' %status)
    return status == 0


def is_launch_agent_valid():
    launchagent_dir = os.path.join(HOME_DIR, 'Library', 'LaunchAgents')
    launchagent_file = os.path.join(launchagent_dir, LAUNCHAGENT_FILENAME)
    file_exists = os.path.isfile(launchagent_file)
    if not file_exists:
        return False

    launchagent = LAUNCHAGENT_TEMPLATE.format(
        label=LAUNCHAGENT_LABEL,
        login_keychain=LOGIN_KEYCHAIN,
        duo_keychain=KEYCHAIN)

    with open(launchagent_file, 'r') as f:
        if launchagent not in f.read():
            print ('=== Launch agent misconfigured.')
            return False
    return True


def configure_launchagent():
    """
    Set up a launchagent to unlock the duo-auth keychain on login
    """
    print('=== Checking launch agent...')
    launchagent_dir = os.path.join(HOME_DIR, 'Library', 'LaunchAgents')
    if not os.path.exists(launchagent_dir):
        os.makedirs(launchagent_dir)
        if not os.path.exists(launchagent_dir):
            print('=== Creating launchagent directory failed.')
            return False

    launchagent_file = os.path.join(launchagent_dir, LAUNCHAGENT_FILENAME)
    launchagent = LAUNCHAGENT_TEMPLATE.format(
        label=LAUNCHAGENT_LABEL,
        login_keychain=LOGIN_KEYCHAIN,
        duo_keychain=KEYCHAIN)

    if is_launch_agent_valid():
        print ('=== Launch agent found')
    else:
        print ('=== Configuring the launchagent to unlock the duo-auth keychain...')
        with open(launchagent_file, 'w') as f:
            f.write(launchagent)

        status = subprocess.call([
            'launchctl', 'load', launchagent_file])
        if status != 0:
            print('=== Loading launch agent failed.')
            return False

    return True


def delete_launchagent():
    """
    Remove the launch agent that attempts to unlock the duo-auth keychain
    """
    print('=== Deleting the launchagent to unlock the duo-auth keychain...')
    launchagent_file = os.path.join(
        HOME_DIR, 'Library', 'LaunchAgents', LAUNCHAGENT_FILENAME)
    if os.path.isfile(launchagent_file):
        subprocess.call(['launchctl', 'unload', launchagent_file])
        os.remove(launchagent_file)

    if is_launch_agent_valid():
        print('=== Deleting launch agent failed.')
        return False
    return True


def configure_duo_keychain():
    """
    Configure the Duo keychain so that the password to the keychain is
    in the user's login keychain. Set up a launchagent to unlock the
    custom keychain on login.
    """
    keychain_password = get_duo_keychain_password()
    if keychain_password:
        print('=== Keychain password found...')
    else:
        print('=== No keychain password found. Creating...')
        keychain_password = generate_duo_keychain_password()
        if not keychain_password:
            print('=== Failed to create keychain password.')
            return False

    keychain_exists = duo_keychain_exists()
    if keychain_exists:
        print('=== Keychain already exists...')
        # Make sure we can successfully unlock the keychain
        lock_duo_keychain()
        keychain_unlocked = unlock_duo_keychain(keychain_password)
        if not keychain_unlocked:
            print('=== Failed to unlock Duo keychain. Recreating Duo keychain...')
            success = all([
                delete_duo_keychain(),
                create_duo_keychain(keychain_password),
                unlock_duo_keychain(keychain_password),
            ])
            if not success:
                return False
    else:
        print('=== Duo keychain not set up. Deleting keychain(if exists) and creating a new keychain.')
        delete_duo_keychain()
        success = create_duo_keychain(keychain_password)
        if not success:
            return False

    return configure_launchagent()


def set_key_partition_list():
    print('=== Setting the partition_id ACL for known browsers and apps...')

    keychain_password = get_duo_keychain_password()
    if not keychain_password:
        print('=== Failed to get duo keychain password.')
        return False

    try:
        acl = 'apple-tool:,apple:,unsigned:'
        global ACL_TEAMIDS
        for teamid in ACL_TEAMIDS:
            acl += ',teamid:' + teamid

        status = subprocess_call(
            as_user([
                'security', 'set-key-partition-list',
                '-l', 'Imported Private Key',
                '-S', acl,
                '-k', keychain_password,
                '-s', KEYCHAIN,
            ]))
    except Exception:
        print('=== set-key-parititon-list does not exist on legacy macOS versions. Continuing...')
    else:
        if status != 0:
            print "=== Failed to set key partition list with status code: %s. Continuing..." %status
    return True


def setup():
    """
    Perform preinstall checks. Check to see if any of the certs we are about
    to install already exist. Check to see if the cert is about to expire.
    If any of these conditions are met, clean up any existing certs and
    proceed with the certificate install. If the cert already exists and
    is not within the expiration period, skip the installation step.
    """
    print('=== Performing preinstall checks...')

    exit_if_running_as_root()
    success = configure_duo_keychain()
    if not success:
        print('=== Preinstall checks failed. Exiting')
        sys.exit(1)

    # Delete any existing Duo certs in the login keychain. We need to do
    # this because we may have existing machines configured with the Duo
    # cert in the login keychain instead of the custom Duo keychain.
    # This could happen if manual enrollement was previously used.
    delete_certs(CERTNAME, LOGIN_KEYCHAIN, error_log_only=True)
    delete_certs(ROOT_CERTNAME, LOGIN_KEYCHAIN, error_log_only=True)
    delete_certs(INTERMEDIATE_CERTNAME, LOGIN_KEYCHAIN, error_log_only=True)
    (cert_hash, root_hash, intermediate_hash, has_multiple_certs) = get_cert_hashes()
    # We are in a bad state if ANY of the leaf, intermediate,
    # and root exists, but not all of them exist.
    clear_existing_certs = has_multiple_certs or (
        any([cert_hash, root_hash, intermediate_hash]) and
            not all([cert_hash, root_hash, intermediate_hash]))

    if cert_hash:
        # Perform expiration check
        should_renew = should_renew_certificate()
        if should_renew:
            clear_existing_certs = True
            print('=== Existing certificate is within expiration period. '
                  'Renewing...')

        # Check for akey/mkey in the cert
        cert_valid = cert_has_required_attributes()
        if not cert_valid:
            clear_existing_certs = True
            print('=== Existing certificate does not have required attributes...')

        if not cert_has_correct_validity_duration():
            clear_existing_certs = True
            print('=== Existing certificate does not have the proper validity duration...')

    # If any of the leaf cert, the root cert, or intermediate cert
    # were somehow removed, clean them up
    if clear_existing_certs:
        print('=== Existing certs in an unclean state. Cleaning and '
              'installing a new certificate...')
        success = delete_duo_keychain_certs()
        if not success:
            print('=== Failed to clean up existing certs')
            sys.exit(1)

    if all([cert_hash, root_hash, intermediate_hash]) and not clear_existing_certs:
        # All certs exist and the cert is not expiring. We can exit early
        # without reinstalling the cert.
        print('=== The cert for the current user already exists. '
              'Skipping installation...')
        # Make sure defaults are set properly. This is an idempotent call
        try:
            set_defaults()
        except Exception as e:
            print('=== Error while setting browser defaults: %s' %e)
            sys.exit(1)

        # We still need to set the partition list in case of an update
        success = set_key_partition_list()
        if not success:
            sys.exit(1)
        else:
            print ('=== All checks complete')
            sys.exit(0)
    else:
        print('=== Preinstall checks complete!')


def get_cert_hashes():
    # Get hashes for any certs that match the common names below. Only set the
    # hash variables if there is EXACTLY 1 certificate matching the CN
    # provided. Otherwise clear the existing cert and install new certs.
    cert_hashes = get_hashes(CERTNAME, KEYCHAIN, require_private_key=True)
    if len(cert_hashes) == 1:
        cert_hash = cert_hashes[0]
    else:
        cert_hash = None

    root_hashes = get_hashes(ROOT_CERTNAME, KEYCHAIN)
    if len(root_hashes) == 1:
        root_hash = root_hashes[0]
    else:
        root_hash = None

    intermediate_hashes = get_hashes(INTERMEDIATE_CERTNAME, KEYCHAIN)
    if len(intermediate_hashes) == 1:
        intermediate_hash = intermediate_hashes[0]
    else:
        intermediate_hash = None

    has_multiple_certs = (len(cert_hashes) > 1 or len(root_hashes) > 1 or
            len(intermediate_hashes) > 1)

    return (cert_hash, root_hash, intermediate_hash, has_multiple_certs)


def generate_private_key():
    """
    Generate a client side private key for signing the certificate signing
    request
    """
    print('=== Generating client side private key...')
    key = crypto.PKey()
    key.generate_key(crypto.TYPE_RSA, KEYSIZE)
    return key


def generate_csr(private_key):
    """
    Generate a certificate signing request for the Duo client authentication
    certificate signed using the provided private key.
    """
    print('=== Generating certificate signing request...')

    req = crypto.X509Req()
    req.get_subject().CN = CERTNAME

    # Add in custom extensions
    custom_extension_template = 'otherName:{oid};UTF8:{data}'
    device_id_oid = '1.3.6.1.4.1.48626.100.3'
    user_id_oid = '1.3.6.1.4.1.48626.100.4'

    extensions = []
    if HARDWARE_UUID:
        extensions.append(custom_extension_template.format(
            oid=device_id_oid, data=HARDWARE_UUID))
    if CONSOLEUSER and COMPUTER_NAME:
        extensions.append(custom_extension_template.format(
            oid=user_id_oid, data=COMPUTER_NAME + '/' + CONSOLEUSER))

    if extensions:
        custom_extension_string = ','.join(extensions)
        x509_extensions = ([
            crypto.X509Extension(
                'subjectAltName', False, custom_extension_string)])
        req.add_extensions(x509_extensions)

    req.set_pubkey(private_key)
    req.sign(private_key, 'sha256')
    csr_pem = base64.b64encode(
        crypto.dump_certificate_request(crypto.FILETYPE_ASN1, req))
    return csr_pem


def get_certificate(csrFile):
    """
    Request CSS to sign the certificate signing request and return
    the signed certificate.
    """
    print('=== Generating client certificate from certificate signing request...')

    # Construct our dict for the JSON body of the enrollment API check_call
    # We have to use an OrderedDict because apparently the key order matters
    body = OrderedDict([
        ('timestamp', datetime.utcnow().isoformat()),
        ('request', OrderedDict([
            ('Flags', 0),
            ('TemplateName', TEMPLATE_NAME),
            ('Pkcs10Request', csrFile)
        ])),
    ])

    # Format the body JSON for use in calculating the HMAC signature
    body_json = json.dumps(body, separators=(',',':')).translate(None, '\r\n')

    # Calculate the HMAC using our secret
    h = hmac.new(APPLICATION_SECRET, body_json, hashlib.sha1).hexdigest()
    # Base64 encode the signature for the headers
    bh = base64.b64encode(h.decode('hex'))

    # Construct our API call request
    clen = len(body_json)
    b64_auth_string = base64.b64encode('%s:%s' % (USERNAME, PASSWORD))
    headers = {
        'Content-Type': 'application/json',
        'Content-Length': clen,
        'Authorization': 'Basic %s' % b64_auth_string,
        'X-CSS-CMS-AppKey': APPLICATION_KEY,
        'X-CSS-CMS-Signature': bh,
        'Content-Type': 'application/json'
    }

    req = urllib2.Request(ENROLLMENT_URL, body_json, headers)

    # Send the request
    f = urllib2.urlopen(req)
    response = json.loads(f.read())
    f.close()

    # Get the returned cert, fix Windows line endings because that's a thing.
    if 'Certificates' not in response:
        raise Exception('Invalid certificate response')
    else:
        pem_data = response['Certificates'].replace('\r\n', '\n')
        cert = '-----BEGIN CERTIFICATE-----\n%s-----END CERTIFICATE-----' % pem_data
        return cert


def write_certificate_to_keychain(certfile, keyfile):
    """
    Install to keychain the signed certificate and the private key generated
    client side
    """
    print('=== Writing keychain entries for client certificate and private key...')
    if (CONSOLEUSER):
        subprocess.check_output([
            'launchctl', 'asuser', CONSOLEUID,
            'security', 'import', keyfile.name,
            '-x', '-A', '-k', KEYCHAIN,])
        subprocess.check_output([
            'launchctl', 'asuser', CONSOLEUID,
            'security', 'import', certfile.name, '-k', KEYCHAIN,])
    else:
        print('=== Running without console user or as root user, '
              'skipping importing key and cert ===')
        raise Exception('Invalid user provided')

def verify_chrome_defaults():
    try:
        res = subprocess.check_output([
            'defaults', 'read', '/Library/Preferences/com.google.Chrome.plist',
            'AutoSelectCertificateForUrls'])
    except Exception:
        try:
            res = subprocess.check_output([
                'defaults', 'read', 'com.google.Chrome',
                'AutoSelectCertificateForUrls'])
        except Exception:
            res = ''
    return 'duosecurity.com/frame' in res

def verify_safari_defaults():
    try:
        res = subprocess.check_output(
            as_user([
                'security', 'get-identity-preference',
                '-s', '*.duosecurity.com', KEYCHAIN,
            ]))
    except Exception as e:
        res = ''

    return 'Duo Device Authentication' in res

def set_defaults():
    """
    Set browser defaults to automatically select the Duo certificate
    for Duo URLs, if not set already.
    """
    print('=== Checking Chrome defaults...')
    if verify_chrome_defaults():
        print('=== Expected Chrome policy already configured. Skipping...')
    else:
        print('=== Setting Chrome defaults for the Duo client certificate...')
        try:
            subprocess.check_output([
                'defaults', 'write',
                '/Library/Preferences/com.google.Chrome.plist',
                'AutoSelectCertificateForUrls', '-array-add', '-string',
                '{\"pattern\":\"https://[*.]duosecurity.com/frame\",\"filter\":{}}'])
        except Exception:
            subprocess.check_output([
                'defaults', 'write', 'com.google.Chrome',
                'AutoSelectCertificateForUrls', '-array-add', '-string',
                '{\"pattern\":\"https://[*.]duosecurity.com/frame\",\"filter\":{}}'])

        # Read it back to make sure it worked
        if not verify_chrome_defaults():
            print('=== It appears setting Chrome defautls failed. User might see Chrome prompts during login. Continue...')


    print('=== Checking Safari identify preference...')
    try:
        res = subprocess.check_output(
            as_user([
                'security', 'get-identity-preference',
                '-s', '*.duosecurity.com', KEYCHAIN,
            ]))
    except Exception as e:
        res = ''

    if verify_safari_defaults():
        print('=== Expected Safari policy already configured. Skipping...')
    else:
        print('=== Setting Safari identify preference for the Duo client certificate...')

        subprocess.check_output(
            as_user([
                'security', 'set-identity-preference',
                '-c', 'Duo Device Authentication',
                '-s', '*.duosecurity.com', KEYCHAIN,
            ]))

        # Read it back to make sure it worked
        if not verify_safari_defaults():
            print('=== It appears setting Safari defautls failed. Safari might not be reported as a Trusted Endpoint. Continue...')


def cleanup(temp_dir):
    print('=== Cleaning up temporary files...')

    keyfile_uri = os.path.join(temp_dir, KEYFILE_NAME)
    try:
        os.remove(keyfile_uri)
    except OSError:
        pass

    try:
        os.rmdir(temp_dir)
    except OSError:
        pass


def delete_duo_keychain_certs():
    return all ([
        delete_certs(CERTNAME, KEYCHAIN),
        delete_certs(ROOT_CERTNAME, KEYCHAIN),
        delete_certs(INTERMEDIATE_CERTNAME, KEYCHAIN),
    ])

def is_valid_cert(cert_hash, root_hash, intermediate_hash):
    """
    Return a True/False value that represents whether or not the proper
    certificates exist on the machine and also are still valid
    """
    if (not all([cert_hash, root_hash, intermediate_hash]) or
            not cert_has_required_attributes()):
        return False

    expiration = get_cert_expiration()
    return datetime.utcnow() < expiration

def create_temp_keychain():
    """
    Create a temporary keychain. Returns True or False if successful.
    """
    print('=== Creating temp keychain...')
    status = subprocess.call(
        as_user([
            'security', 'create-keychain',
            '-p', TEMP_KEYCHAIN_PASSWORD, TEMP_KEYCHAIN,
        ]))
    if status != 0:
        print ('=== Create temp keychain failed with status: %s' % status)
        return False
    return True


def delete_temp_keychain():
    """
    Delete the temporary keychain. Returns True or False if successful.
    """
    print('=== Deleting temp keychain...')
    status = subprocess.call(
        as_user(['security', 'delete-keychain', TEMP_KEYCHAIN]))
    if status != 0:
        print ('=== Delete temp keychain failed with status: %s' % status)
        return False
    return True


def trigger_keychain_event():
    """
    Create and delete a temporary keychain in order to trigger a
    keychainListChangedEvent so that Chrome refreshes its list of certs.
    """
    try:
        created = create_temp_keychain()
        if created:
            delete_temp_keychain()
    except Exception as e:
        print ('=== Error triggering keychain event... Error: %s' % e)


def main():
    try:
        init_consts()
    except Exception as e:
        print('=== Error setting up initial constant variables... Error: %s' % e)
        sys.exit(1)

    if RUNNING_IN_DELETE_MODE:
        print('=== Deleting Duo certificates while running in DELETE MODE...')
        success = all([
            delete_duo_keychain_certs(),
            delete_duo_keychain(),
            delete_launchagent(),
        ])
        if not success:
            sys.exit(1)

        sys.exit(0)

    if RUNNING_IN_CERT_CHECK_MODE:
        # Make sure we have a valid cert
        # Make sure we can access this cert by unlocking the Duo keychain
        # Make sure this happens automatically by checking the launch agent.
        print('=== Checking for Duo certs while in CERT CHECK MODE...')

        (cert_hash, root_hash, intermediate_hash, _) = get_cert_hashes()
        has_valid_cert = is_valid_cert(cert_hash, root_hash, intermediate_hash)

        keychain_password = get_duo_keychain_password()
        keychain_exists = duo_keychain_exists()
        keychain_unlocked = False
        if keychain_password and keychain_exists:
            lock_duo_keychain()
            keychain_unlocked = unlock_duo_keychain(keychain_password)

        print('<result>{}</result>'.format(all([
            has_valid_cert,
            keychain_unlocked,
            is_launch_agent_valid(),
        ])))
        sys.exit(0)

    try:
        # Perform preinstall checks
        setup()

        # Generate a client side private key and sign the certificate signing
        # request with the private key
        private_key = generate_private_key()
        csr_pem = generate_csr(private_key)

        # Send the CSR to CSS and get a valid client certificate
        cert_pem = get_certificate(csr_pem)
    except Exception as e:
        print('=== Error during certificate generation... Error: %s' % e)
        sys.exit(1)

    temp_dir = tempfile.mkdtemp()
    try:
        keyfile_uri = os.path.join(temp_dir, KEYFILE_NAME)
        with tempfile.NamedTemporaryFile(
                mode='w', suffix='.pem') as certfile, open(
                keyfile_uri, 'w') as keyfile:
            certfile.write(cert_pem)
            certfile.flush()
            keyfile.write(
                crypto.dump_privatekey(crypto.FILETYPE_PEM, private_key))
            keyfile.flush()
            write_certificate_to_keychain(certfile, keyfile)

        try:
            set_defaults()
        except Exception as e:
            print('=== Error while setting browser defaults: %s' %e)
            cleanup(temp_dir)
            sys.exit(1)

        success = set_key_partition_list()
        if not success:
            cleanup(temp_dir)
            sys.exit(1)

        trigger_keychain_event()

        print('=== Cert installation complete!')
        cleanup(temp_dir)
        sys.exit(0)
    except Exception as e:
        print('=== Error during certificate installation... Error: %s' % e)
        cleanup(temp_dir)
        sys.exit(1)


if __name__ == '__main__':
    main()
e here]

Optionally include the parameter -f to make this script overwrite existing .ps1 files.

Example:
>python ps1scriptify.py foo.py -f

Will create a file named Foo.ps1, overwriting it if it already exists. 

Or, you can specify the destination folder by using the -dest parameter.

Example:
>python ps1scriptify.py foo.py -dest C:\bar

Will create a file named Foo.ps1 in folder C:\bar.

Fun fact: you can test this script by running it on itself!

