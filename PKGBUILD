# Maintainer: Yeqin Su <hougelangley1987@gmail.com>
## Just For GCC ##
## Delete Useless thing ##

pkgbase=linux-xanmod-cacule-uksm
pkgver=5.12.10
_major=5.12
_branch=5.x
xanmod=1
pkgrel=${xanmod}
pkgdesc='Linux Xanmod. Branch with Cacule scheduler by Hamad Marri'
url="http://www.xanmod.org/"
arch=(x86_64)

license=(GPL2)
makedepends=(
    xmlto kmod inetutils bc libelf cpio)
options=('!strip')
_srcname="linux-${pkgver}-xanmod${xanmod}"

source=("https://cdn.kernel.org/pub/linux/kernel/v${_branch}/linux-${_major}.tar."{xz,sign}
        "https://github.com/HougeLangley/customkernel/releases/download/v5.12-patch/patch-5.12.10-xanmod1-cacule"
        'v1-cjktty.patch'
        'v1-uksm.patch')
validpgpkeys=(
        'ABAF11C65A2970B130ABE3C479BE3E4300411886' # Linux Torvalds
        '647F28654894E3BD457199BE38DBBDC86092693E' # Greg Kroah-Hartman
         )

# Archlinux patches
_commits=""
for _patch in $_commits; do
source+=("${_patch}.patch::https://git.archlinux.org/linux.git/patch/?id=${_patch}")
done

sha256sums=('7d0df6f2bf2384d68d0bd8e1fe3e071d64364dcdc6002e7b5c87c92d48fac366'
        '39045607567d69f84424b224e4fa6bf8f97a21a06ac9d6396acab16a18c4bcd3'
        'SKIP'
        'SKIP'
        'SKIP')

export KBUILD_BUILD_HOST=${KBUILD_BUILD_HOST:-archlinux}
export KBUILD_BUILD_USER=${KBUILD_BUILD_USER:-makepkg}
export KBUILD_BUILD_TIMESTAMP=${KBUILD_BUILD_TIMESTAMP:-$(date -Ru${SOURCE_DATE_EPOCH:+d @$SOURCE_DATE_EPOCH})}

prepare() {
    cd linux-${_major}

# Apply Xanmod patch
    patch -Np1 -i ../patch-${pkgver}-xanmod${xanmod}-cacule

    msg2 "Setting version..."
    scripts/setlocalversion --save-scmversion
    echo "-$pkgrel" > localversion.10-pkgrel
    echo "${pkgbase#linux-xanmod}" > localversion.20-pkgname

# Archlinux patches
    local src
    for src in "${source[@]}"; do
        src="${src%%::*}"
        src="${src##*/}"
        [[ $src = *.patch ]] || continue
        msg2 "Applying patch $src..."
        patch -Np1 < "../$src"
    done

  # Applying configuration
  cp -vf CONFIGS/xanmod/gcc/config .config
  # enable LTO_CLANG_THIN
  if [ "${_compiler}" = "clang" ]; then
    scripts/config --disable LTO_CLANG_FULL
    scripts/config --enable LTO_CLANG_THIN
    _LLVM=1
  fi

  # CONFIG_STACK_VALIDATION gives better stack traces. Also is enabled in all official kernel packages by Archlinux team
  scripts/config --enable CONFIG_STACK_VALIDATION

  # Enable IKCONFIG following Arch's philosophy
  scripts/config --enable CONFIG_IKCONFIG \
                 --enable CONFIG_IKCONFIG_PROC

  # User set. See at the top of this file
  if [ "$use_tracers" = "n" ]; then
    msg2 "Disabling FUNCTION_TRACER/GRAPH_TRACER only if we are not compiling with clang..."
    if [ "${_compiler}" = "gcc" ]; then
      scripts/config --disable CONFIG_FUNCTION_TRACER \
                     --disable CONFIG_STACK_TRACER
    fi
  fi

  if [ "$use_numa" = "n" ]; then
    msg2 "Disabling NUMA..."
    scripts/config --disable CONFIG_NUMA
  fi

  # Let's user choose microarchitecture optimization in GCC
  # sh ${srcdir}/choose-gcc-optimization.sh $_microarchitecture

  # This is intended for the people that want to build this package with their own config
  # Put the file "myconfig" at the package folder (this will take preference) or "${XDG_CONFIG_HOME}/linux-xanmod/myconfig"
  # If we detect partial file with scripts/config commands, we execute as a script
  # If not, it's a full config, will be replaced
  for _myconfig in "${SRCDEST}/myconfig" "${HOME}/.config/linux-xanmod/myconfig" "${XDG_CONFIG_HOME}/linux-xanmod/myconfig" ; do
    if [ -f "${_myconfig}" ] && [ "$(wc -l <"${_myconfig}")" -gt "0" ]; then
      if grep -q 'scripts/config' "${_myconfig}"; then
        # myconfig is a partial file. Executing as a script
        msg2 "Applying myconfig..."
        bash -x "${_myconfig}"
      else
        # myconfig is a full config file. Replacing default .config
        msg2 "Using user CUSTOM config..."
        cp -f "${_myconfig}" .config
      fi
      echo
      break
    fi
  done

  ### Optionally load needed modules for the make localmodconfig
  # See https://aur.archlinux.org/packages/modprobed-db
  if [ "$_localmodcfg" = "y" ]; then
    if [ -f $HOME/.config/modprobed.db ]; then
      msg2 "Running Steven Rostedt's make localmodconfig now"
      make LLVM=$_LLVM LLVM_IAS=$_LLVM LSMOD=$HOME/.config/modprobed.db localmodconfig
    else
      msg2 "No modprobed.db data found"
      exit
    fi
  fi

  make LLVM=$_LLVM LLVM_IAS=$_LLVM olddefconfig

  make -s kernelrelease > version
  msg2 "Prepared %s version %s" "$pkgbase" "$(<version)"

  [[ -z "$_makenconfig" ]] || make LLVM=$_LLVM LLVM_IAS=$_LLVM nconfig

  # save configuration for later reuse
  cat .config > "${SRCDEST}/config.last"
}

build() {
	cd linux-${_major}
	make all -j16
}

_package() {
	pkgdesc="The Linux kernel and modules with Xanmod patches"
    depends=(coreutils kmod initramfs)
    optdepends=('crda: to set the correct wireless channels of your country'
            'linux-firmware: firmware images needed for some devices')

    cd linux-${_major}
	local kernver="$(<version)"
    local modulesdir="$pkgdir/usr/lib/modules/$kernver"

    msg2 "Installing boot image..."
# systemd expects to find the kernel here to allow hibernation
# https://github.com/systemd/systemd/commit/edda44605f06a41fb86b7ab8128dcf99161d2344
    install -Dm644 "$(make -s image_name)" "$modulesdir/vmlinuz"

# Used by mkinitcpio to name the kernel
    echo "$pkgbase" | install -Dm644 /dev/stdin "$modulesdir/pkgbase"

    msg2 "Installing modules..."
    make INSTALL_MOD_PATH="$pkgdir/usr" INSTALL_MOD_STRIP=1 modules_install

  # remove build and source links
  rm "$modulesdir"/{source,build}
}

_package-headers() {
	pkgdesc="Header files and scripts for building modules for Xanmod Linux kernel"
    depends=(pahole)

    cd linux-${_major}
	local builddir="$pkgdir/usr/lib/modules/$(<version)/build"

    msg2 "Installing build files..."
    install -Dt "$builddir" -m644 .config Makefile Module.symvers System.map \
    localversion.* version vmlinux
    install -Dt "$builddir/kernel" -m644 kernel/Makefile
    install -Dt "$builddir/arch/x86" -m644 arch/x86/Makefile
    cp -t "$builddir" -a scripts

# add objtool for external module building and enabled VALIDATION_STACK option
    install -Dt "$builddir/tools/objtool" tools/objtool/objtool

# add xfs and shmem for aufs building
    mkdir -p "$builddir"/{fs/xfs,mm}

    msg2 "Installing headers..."
    cp -t "$builddir" -a include
    cp -t "$builddir/arch/x86" -a arch/x86/include
    install -Dt "$builddir/arch/x86/kernel" -m644 arch/x86/kernel/asm-offsets.s

    install -Dt "$builddir/drivers/md" -m644 drivers/md/*.h
    install -Dt "$builddir/net/mac80211" -m644 net/mac80211/*.h

# http://bugs.archlinux.org/task/13146
    install -Dt "$builddir/drivers/media/i2c" -m644 drivers/media/i2c/msp3400-driver.h

# http://bugs.archlinux.org/task/20402
    install -Dt "$builddir/drivers/media/usb/dvb-usb" -m644 drivers/media/usb/dvb-usb/*.h
    install -Dt "$builddir/drivers/media/dvb-frontends" -m644 drivers/media/dvb-frontends/*.h
    install -Dt "$builddir/drivers/media/tuners" -m644 drivers/media/tuners/*.h

    msg2 "Installing KConfig files..."
    find . -name 'Kconfig*' -exec install -Dm644 {} "$builddir/{}" \;

    msg2 "Removing unneeded architectures..."
    local arch
    for arch in "$builddir"/arch/*/; do
              [[ $arch = */x86/ ]] && continue
    echo "Removing $(basename "$arch")"
    rm -r "$arch"
    done

    msg2 "Removing documentation..."
    rm -r "$builddir/Documentation"

    msg2 "Removing broken symlinks..."
    find -L "$builddir" -type l -printf 'Removing %P\n' -delete

    msg2 "Removing loose objects..."
    find "$builddir" -type f -name '*.o' -printf 'Removing %P\n' -delete

    # msg2 "Stripping build tools..."
    # local file
    # while read -rd '' file; do
        # case "$(file -bi "$file")" in
            # application/x-sharedlib\;*)      # Libraries (.so)
                # strip -v $STRIP_SHARED "$file" ;;
            # application/x-archive\;*)        # Libraries (.a)
                # strip -v $STRIP_STATIC "$file" ;;
            # application/x-executable\;*)     # Binaries
                # strip -v $STRIP_BINARIES "$file" ;;
            # application/x-pie-executable\;*) # Relocatable binaries
                # strip -v $STRIP_SHARED "$file" ;;
        # esac
    # done < <(find "$builddir" -type f -perm -u+x ! -name vmlinux -print0)

    msg2 "Adding symlink..."
    mkdir -p "$pkgdir/usr/src"
    ln -sr "$builddir" "$pkgdir/usr/src/$pkgbase"
}

pkgname=("${pkgbase}" "${pkgbase}-headers")
for _p in "${pkgname[@]}"; do
eval "package_$_p() {
	$(declare -f "_package${_p#$pkgbase}")
		_package${_p#$pkgbase}
}"
done
