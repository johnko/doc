<div id="sites-canvas">
<div id="goog-ws-editor-toolbar-container"> </div>
<div xmlns="http://www.w3.org/1999/xhtml" id="title-crumbs" style="">
<a dir="ltr" href="/software-development">Software Development</a>‎ &gt; ‎<a dir="ltr" href="/software-development/articles">Articles</a>‎ &gt; ‎
  </div>
<h3 xmlns="http://www.w3.org/1999/xhtml" id="sites-page-title-header" style="" align="left">
<span id="sites-page-title" dir="ltr">Creating and Using a FreeBSD mfsroot</span>
</h3>
<span class="announcementsPostTimestamp" id="afterPageTitleHideDuringEdit">
<script xmlns="http://www.w3.org/1999/xhtml" type="text/javascript">
      //<![CDATA[
        function JOT_insertRelDate(timestamp, absTimeStr, isSiteLocale, dir) {
          var relTimeStr = JOT_formatRelativeToNow(timestamp, isSiteLocale);
          if (relTimeStr) {
            if (isSiteLocale) {
              document.write('<span timestamp="' + timestamp + '" issitelocale="' + isSiteLocale +
                '" title="' + absTimeStr + '" dir="' + dir + '">' + relTimeStr + '<' + '/span>');
            } else {
              document.write('<span timestamp="' + timestamp + '" title="' + absTimeStr +
              '" dir="' + dir + '">' + relTimeStr + '<' + '/span>');
            }
          } else {
            document.write(absTimeStr);
          }
        }
      //]]>
</script>
    
  
  posted <span xmlns="http://www.w3.org/1999/xhtml" dir="ltr">Dec 31, 2009, 4:19 PM</span> by Michael Miranda - A

                  
                      &nbsp;
                      <span id="sites-announcement-updated-time" class="updatedTime">
                        [
  
  
  
      <script xmlns="http://www.w3.org/1999/xhtml" type="text/javascript">
      //<![CDATA[
        function JOT_insertRelDate(timestamp, absTimeStr, isSiteLocale, dir) {
          var relTimeStr = JOT_formatRelativeToNow(timestamp, isSiteLocale);
          if (relTimeStr) {
            if (isSiteLocale) {
              document.write('<span timestamp="' + timestamp + '" issitelocale="' + isSiteLocale +
                '" title="' + absTimeStr + '" dir="' + dir + '">' + relTimeStr + '<' + '/span>');
            } else {
              document.write('<span timestamp="' + timestamp + '" title="' + absTimeStr +
              '" dir="' + dir + '">' + relTimeStr + '<' + '/span>');
            }
          } else {
            document.write(absTimeStr);
          }
        }
      //]]>
</script>
    
  
  updated <span xmlns="http://www.w3.org/1999/xhtml" dir="ltr">Jan 30, 2010, 5:25 PM</span>
]
                      </span>
</span>
<div id="sites-canvas-main" class="sites-canvas-main">
<div id="sites-canvas-main-content">
<table xmlns="http://www.w3.org/1999/xhtml" cellspacing="0" class="sites-layout-name-one-column sites-layout-hbox"><tbody><tr><td class="sites-layout-tile sites-tile-name-content-1">
<div dir="ltr"><cite>by Michael J. A. Miranda, December 28, 2007</cite>
<h2><a name="TOC-Purpose-of-an-mfsroot"></a>Purpose of an mfsroot</h2>
<p>FreeBSD has a means to run with a file system that resides fully in memory (RAM). The primary benefits of such a system are: </p>
<ul>
<li>By running in memory, the operating system operates at a faster speed.</li>
<li>Corruption of the operating system's filesystem in memory can be "repaired" by simply rebooting the machine with the original filesystem.</li>
<li>You can create an "embedded system", "firmware" or "appliance" type of machine that can be updated on the fly by simply replacing the filesystem file (mfsroot file) that gets loaded and executed in memory (similar to network routers and switches)</li>
<li>You can create several versions of the operating system's filesystem and switch them out easily as necessary</li></ul>
<h2><a name="TOC-What-does-it-look-like-on-the-hard-drive"></a>What does it look like on the hard drive</h2>
<p>One of the first hurdles to successfully completing a project like this is understanding what the final product will look like. In this case, we are going to end up with a bootable hard drive where the filesystem is contained in only one file. For FreeBSD, this means that the actual bootable hard drive (or other boot device such as usb drive or compact flash card) filesystem will look like this on the bootable partition:</p><code>/boot<br>&nbsp;&nbsp;/kernel<br>&nbsp;&nbsp;&nbsp;&nbsp;/kernel<br>&nbsp;&nbsp;/loader.conf<br>&nbsp;&nbsp;/[...other standard boot files]<br>/mfsroot.gz<br></code>
<p>That is it.</p>
<p>The first question is: Where are all the operating system files and directories? Where are <code>/usr /usr/local/ /etc /bin /sbin ....etc.</code>?</p>
<p>The answer is: All operating system files are in the <code>mfsroot</code> file (<code>mfsroot.gz</code> compressed). </p>
<p>The mfsroot file contains the entire file system excluding the <code>/boot</code> directory and associated files.</p>
<h2><a name="TOC-Creating-an-mfsroot-file"></a>Creating an mfsroot file</h2>
<h3><a name="TOC-Basic-Steps"></a>Basic Steps</h3>
<ol>
<li>Create a filesystem that contains all of the FreeBSD components, programs and ports that you require.</li>
<li>Create an mfsroot image file of a size that can contain the filesystem.</li>
<li>Mount the mfsroot file as a memory disk so that it is available and operational as a disk device.</li>
<li>Copy the filesystem into the mounted mfsroot memory disk.</li>
<li>Unmount the mfsroot memory disk and close the mfsroot image file.</li>
<li>Compress the mfsroot image file using gzip.</li></ol>
<h3><a name="TOC-Short-of-the-Long"></a>Short of the Long</h3>
<h4><a name="TOC-Create-a-filesytem"></a>Create a filesytem</h4>
<p>The best way to create and test the filesystem that you are going to use for the mfsroot is by utilizing FreeBSD's <i>jail</i> system. Use the jail system to install another instance of FreeBSD on your current FreeBSD machine. <i>chroot</i> into the jailed system and install all the necessary packages. Test the filesystem until it meets your requirements. (Google "FreeBSD jail" for more information)</p>
<h4><a name="TOC-Create-an-mfsroot-image-file"></a>Create an mfsroot image file</h4>
<ul>
<li>[output_file] = name of the mfsroot file (usually <i>mfsroot</i>)</li>
<li>[block_size] = size of the blocks of memory in bytes</li>
<li>[number_of_secotrs] = number of blocks of memory that is at least as large as the size of the filesystem</li></ul>
<p>To determine [block_size] and [number_of_sectors] you must first know how big your filesystem is. Execute <code>du -h [path_to_jail_filesytem]</code> to obtain the number of megabytes that the filesystem occupies. Next, use the following conversion table.</p>
<p>Number of Megabytes * 1024 = Number of Kilobytes<br>Number of Kilobytes * 1024 = Number of Bytes<br>[number_sectors] = Number of Bytes / [block_size]<br><br>Decide on a [block_size]. Usually, 512 bytes is used.<br><br>For example, a 120 megabyte file system = 120 X 122,880 kilobytes. (122,880 * 1024) / 512 block size = 245,760 sectors. </p><code>dd if=/dev/zero of=[output_file] bs=[block_size] count=[number_of_sectors]</code>
<h4><a name="TOC-Mounting-the-mfsroot-image-file"></a>Mounting the mfsroot image file</h4>
<p>Prior to copying the filesystem into the mfsroot image file, you must mount the image file to make it accessible like a disk device. This can be accomplished by creating a memory disk in RAM and mounting it.</p><code>mdconfig -a -t vnode -f [output_file]</code>
<p>Example:</p><code>mdconfig -a -t vnode -f mfsroot</code>
<p>This command will return a memory disk device name (i.e. md0, md1). This is the device you need to mount in order to access the mfsroot image file as a disk</p><code>mount /dev/md0 /mnt</code>
<p>The above command essentially mounts the mfsroot image file to the <code>/mnt</code> directory. All you need to do now is copy your filesystem to the <code>/mnt</code> directory to have it added to the mfsroot image file.</p>
<h4><a name="TOC-Copying-the-filesystem-to-the-mfsroot-mounted-image"></a>Copying the filesystem to the mfsroot mounted image</h4>
<p>When you copy the filesystem, you will want to preserve the owner and permission settings of all the files. The best way to do this is by using <code>tar</code>.</p><code>tar cf - [filesystem_root] | tar -C [target_directory] -xfp -</code>
<p>Example:</p><code>tar cf - /usr/jail/myfilesystem | tar -C /mnt -xfp -</code>
<h4><a name="TOC-Umounting-the-mfsroot-image-file"></a>Umounting the mfsroot image file</h4>
<p>Close the mfsroot image file by simply unmounting the memory disk device.</p><code>umount /dev/md0</code>
<h4><a name="TOC-Compress-the-mfsroot-image-file"></a>Compress the mfsroot image file</h4><code>gzip -9 [output_file]</code>
<p>Example:</p><code>gzip -9 mfsroot</code>
<h2><a name="TOC-MFSROOT-Filesystem-Size"></a>MFSROOT Filesystem Size</h2>
<p>The size of your mfsroot file system (the filesystem you built using a <code>jail</code> process) will determine whether you need to make any modifications to the default kernel. First of all, you must consider that the mfsroot filesystem will be loaded into memory upon boot up. Therefore, you need to make sure the destination machine has sufficient memory to hold the filesystem and run the applications. For an embedded/firmware/appliance type of system you would prefer to have a filesystem that is as small as possible. However, if your memory availability and resource needs allow, almost any size could be accommodated. If you build and use the GENERIC kernel, your mfsroot filesystem should be no larger than 100 megabytes to allow it to be loaded by the kernel. If it is larger than that, your the mfsroot filesystem will not load and your machine will continually cycle reboots. In which case, you will need to modify the kernel options to accommodate the larger mfsroot file size.</p>
<h2><a name="TOC-Build-a-Kernel"></a>Build a Kernel</h2>
<p>The kernel and the other files required to boot the system is located outside of the <code>mfsroot</code> file. The best practice is to build a custom kernel to use with your mfsroot-based FreeBSD system. For instructions on how to build a kernel, go to the following link:</p>
<p><a href="http://www.freebsd.org/doc/en_US.ISO8859-1/books/handbook/kernelconfig-building.html" rel="nofollow" target="_blank">http://www.freebsd.org/doc/en_US.ISO8859-1/books/handbook/kernelconfig-building.html</a></p>
<p><b>IMPORTANT NOTE: DO NOT INSTALL THE KERNEL ON YOUR SYSTEM. Simply use this process to build and compile a kernel to use for the mfsroot-based FreeBSD system you are building.</b></p>
<h3><a name="TOC-Large-MFSROOT-Filesystems-Require-Kernel-Source-Code-Changes"></a>Large MFSROOT Filesystems Require Kernel Source Code Changes</h3>
<p>Although the goal when builing an embedded/firmware/appliance system is to reduce the footprint of the filesystem in memory, sometimes business requirements dictate a filesystem size larger than what is allowed by the default kernel configuration. To allow the kernel to loadlarger <code>mfsroot</code> filesystems, you will need to make the following changes to the kernal source code:</p>
<ol>
<li>Find and open for editing the <code>/usr/src/syst/i386/include/pmap.h</code> file.</li>
<li>Find the following variable: <code>NKPT</code> and increase its value.</li>
<li>Find the following variable: <code>KVA_PAGES</code> and increase its value.</li>
<li>Save and exit the <code>pmap.h</code> file and rebuild the kernel as described above.</li></ol>
<p>The most obvious question is: How much do I increase the value of those variables? See the formula below</p>
<h4><a name="TOC-NKPT-and-KVA_PAGES-Formulas"></a>NKPT and KVA_PAGES Formulas</h4>
<p>To Be Developed</p>
<h2><a name="TOC-Using-an-mfsroot-file"></a>Using an mfsroot file</h2>
<p>To Be Developed</p></div></td></tr></tbody></table>
</div> 
</div> 
<div id="sites-canvas-bottom-panel">
<div id="sites-attachments-container">
</div>
<a xmlns="http://www.w3.org/1999/xhtml" name="page-comments"></a>
<div xmlns="http://www.w3.org/1999/xhtml" id="COMP_page-comments"><div class="sites-comment-docos-wrapper"><div class="sites-comment-docos"><div class="sites-comment-docos-background"></div><div class="sites-comment-docos-header"><div class="sites-comment-docos-header-title">Comments</div></div><div id="sites-comment-docos-pane" class="sites-comment-docos-pane" dir="ltr" style="text-align: left;"><div class="dcs-a dcs-a-dcs-cb-dcs-gf dcs-a-dcs-qb" tabindex="0"><div class="dcs-a-dcs-cb-dcs-y"><div class="dcs-a-dcs-cb-dcs-eb dcs-re-dcs-pb-dcs-qb"><img class="dcs-a-dcs-o g-hovercard" width="48" height="48" alt="" id="dcs-img-0" src="//ssl.gstatic.com/s2/profiles/images/silhouette96.png" data-userid="PREF_118280632585120988877"><div class="dcs-a-dcs-cb-dcs-eb-dcs-y"><div class="dcs-a-dcs-cb-dcs-bg dcs-r-dcs-qg-dcs-rg g-hovercard" data-userid="PREF_118280632585120988877">Anonymous</div><div class="dcs-a-dcs-c dcs-a-dcs-cb-dcs-c-dcs-d"><textarea class="dcs-a-dcs-c-dcs-ce" x-webkit-speech="" speech="" aria-haspopup="true" aria-label="Comment draft"></textarea><div class="dcs-a-dcs-c-dcs-t-dcs-ub"></div><div class="dcs-a-dcs-c-dcs-t-dcs-u"></div><div class="dcs-a-dcs-c-dcs-xb-dcs-yb-dcs-nb" style="display: none">Your +mention will add people to this post and send an email.</div><div class="dcs-a-dcs-c-dcs-lb-dcs-mb-dcs-nb" style="display: none">Making sure people you mentioned have access…</div><div class="dcs-a-dcs-c-dcs-wc"><div role="button" class="dcs-r-dcs-qg-dcs-rg dcs-j-dcs-l dcs-j-dcs-l-dcs-dc dcs-a-dcs-c-dcs-sf" tabindex="0" title="Post comment" style="-webkit-user-select: none;"><span class="dcs-a-dcs-c-dcs-wc-dcs-sf">Comment</span></div><div role="button" class="dcs-r-dcs-qg-dcs-rg dcs-j-dcs-l dcs-j-dcs-l-dcs-ke dcs-a-dcs-c-dcs-kb" tabindex="0" title="Discard comment" style="-webkit-user-select: none;">Cancel</div></div></div></div></div><div class="dcs-a-dcs-cb-dcs-vb dcs-ob-dcs-pb-dcs-qb dcs-r-dcs-qg-dcs-rg">You do not have permission to add comments.</div><div class="dcs-a-dcs-qb dcs-a dcs-a-dcs-uc-dcs-vc"></div></div><div></div></div><span tabindex="0" style="position: absolute;"></span></div></div></div></div>
</div>
</div>
