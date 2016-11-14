#!/bin/bash

#
# Copyright (c) 2015, Erigones, s. r. o. All rights reserved.
#

. /lib/svc/share/fs_include.sh

export PATH="/usr/sbin:/sbin:/usr/bin"

USB_MNT="/mnt/usbkey"
USB_CONFIG="${USB_MNT}/config"
PW=

declare BOOTED_FROM_HDD

fatal()
{
    echo "$*" >&2
    exit 1
}

mount_usb()
{
    if mount | grep "^${USB_MNT}" >/dev/null 2>&1; then
        USB_ALREADY_MOUNTED=true
        return 0
    fi

    USBKEYS=$(disklist -a)
    for key in ${USBKEYS}; do
        if [[ $(fstyp /dev/dsk/${key}p0:1) == 'pcfs' ]]; then
            if mount -F pcfs -o foldcase,noatime /dev/dsk/${key}p0:1 ${USB_MNT}; then
                if [[ ! -f ${USB_MNT}/.joyliveusb ]]; then
                    umount ${USB_MNT};
                else
                    break;
                fi
            fi
        fi
    done

    if mount | grep "^${USB_MNT}" >/dev/null 2>&1; then
        return 0
    fi

    if [[ ! -f ${USB_CONFIG} ]]; then
        return 1
    fi
}

umount_usb() {
    if [[ -n "${USB_ALREADY_MOUNTED}" ]]; then
        return 0
    fi

    if mount | grep "^${USB_MNT}" >/dev/null 2>&1; then
        umount ${USB_MNT}
    fi
}

prompt_root_password()
{
                read -s -p "New Password: " pw1
                echo
                read -s -p "Re-enter new Password: " pw2
                echo

                if [[ "$pw1" != "$pw2" ]]; then
                        fatal "Passwords don't match."
                else
                        PW=${pw1}
                fi
}

set_root_password()
{
    local enc_password=$1

    # First, we change the password on the USB key, then we change it
    # for the running system.
    if /bin/bootparams | grep "^headnode=true" > /dev/null 2>&1; then
      if [ ${BOOTED_FROM_HDD} -eq 0 ]; then
          (mount_usb \
          && sed -e "s|^root_shadow=.*|root_shadow=\'${enc_password}\'|" ${USB_CONFIG} > /tmp/.config.$$ \
          && mv /tmp/.config.$$ ${USB_CONFIG} \
          && umount_usb) || fatal "Failed to change root password on the USB key"
      fi

      sed -e "s|^root:[^\:]*:|root:${enc_password}:|" /etc/shadow > /etc/shadow.new \
      && chmod 400 /etc/shadow.new \
      && mv /etc/shadow.new /etc/shadow
    fi

    # Compute node is a different story because of /etc/shadow lofs mount.
    if /bin/bootparams | grep "^smartos=true" > /dev/null 2>&1; then
      if [ ${BOOTED_FROM_HDD} -eq 0 ]; then
          sed -e "s|^root:[^\:]*:|root:${enc_password}:|" /usbkey/shadow > /usbkey/shadow.new \
          && chmod 400 /usbkey/shadow.new \
          && mv /usbkey/shadow.new /usbkey/shadow \
          && umount /etc/shadow \
          && mount -F lofs /usbkey/shadow /etc/shadow
      else
          sed -e "s|^root:[^\:]*:|root:${enc_password}:|" /etc/shadow > /etc/shadow.new \
          && chmod 400 /etc/shadow.new \
          && mv /etc/shadow.new /etc/shadow
      fi
    fi
}

BOOTED_FROM_HDD=1
mounted "/" "-" "zfs" < /etc/mnttab || BOOTED_FROM_HDD=0

prompt_root_password

ENC_PW="$(/usr/lib/cryptpass "${PW}")"
set_root_password "${ENC_PW}" && echo "Password successfully changed." && exit 0
