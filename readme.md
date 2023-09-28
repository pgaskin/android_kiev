```bash
repo init -u https://github.com/LineageOS/android.git -b lineage-18.1 --git-lfs
cat << 'EOF' > .repo/local_manifests/kiev.xml
<?xml version="1.0" encoding="UTF-8"?>
<manifest>
  <project path="device/motorola/kiev" remote="github" name="pgaskin/android_kiev" revision="android_device_motorola_kiev/lineage-18.1" />
  <project path="kernel/motorola/sm7250" remote="github" name="pgaskin/android_kiev" revision="android_kernel_motorola_sm7250/lineage-18.1" />
  <project path="vendor/motorola/kiev" remote="github" name="pgaskin/android_kiev" revision="android_vendor_motorola_kiev/lineage-18.1" />
  <project path="hardware/motorola" remote="github" name="pgaskin/android_kiev" revision="android_hardware_motorola/lineage-18.1" />
</manifest>
EOF
repo sync -j8 --force-sync

source build/envsetup.sh

mm clean
breakfast kiev
mka target-files-package otatools

vars="$(cat out/dumpvars-soong.log | grep -E ' build/soong/ui/build/dumpvars\.go:[0-9]+: [A-Z]' | cut -d " " -f4- | sed 's/ /=/' | sort -u)"
vers="lineage-$(echo "$vars" | grep "^LINEAGE_VERSION" | cut -d= -f2-)"

sign_target_files_apks -o -d ~/keys/signing $OUT/obj/PACKAGING/target_files_intermediates/*-target_files-*.zip $vers-target.zip
img_from_target_files signed-target_files.zip $vers-fastbootd.zip
ota_from_target_files -k ~/keys/signing/releasekey --block --backup=true signed-target_files.zip $vers.zip

{
    printf '**dumpvars**\n\n';
    printf '```\n%s\n```\n\n' "$vars";
    printf '**keys**\n\n';
    printf '```\n';
    for x in ~/keys/signing/*.x509.pem; do printf "%s %s\n" "$(echo ${x##*/} | cut -d. -f1)" "$(openssl x509 -serial -noout -in $x)"; done
    printf '```\n\n';
    printf '**repo**\n\n';
    printf "| %s | %s | %s |\n" "Path / Project" "Commit / Date" "Message" "---" "---" "---";
    repo forall -c '
        c_path="$REPO_INNERPATH";
        c_project="$REPO_PROJECT ([$REPO_REMOTE]($(git remote get-url $REPO_REMOTE)))";
        c_commit="\`$(git log -1 --format="%h")\` ($REPO_RREV)";
        c_date="$(git log -1 --format="%ci")";
        c_message="$(git log -1 --format="%s")";
        if [[ $REPO_REMOTE == github ]]; then c_commit="[\`$(git log -1 --format="%h")\`]($(git remote get-url github)/commits/$(git log -1 --format="%H")) ($REPO_RREV)"; fi;
        if [[ $REPO_REMOTE == aosp ]]; then c_commit="[\`$(git log -1 --format="%h")\`]($(git remote get-url aosp)/+log/$(git log -1 --format="%H")) ($REPO_RREV)"; fi;
        printf "| **%s** <br> %s | %s <br> %s | %s |\n" "$c_path" "$c_project" "$c_commit" "$c_date" "$c_message";
    ' | cat -;
} > $vers.md
```
