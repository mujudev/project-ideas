---------------------------------------------------------------------------
Lunar Terminator Visualization Tool Distribution Folder (Rev. June 7, 2008)
---------------------------------------------------------------------------

This folder contains a release version of LTVT downloaded from:

 http://ltvt.wikispaces.com/

To install the program, simply drag the (uncompressed) folder to any 
convenient location on your hard drive.

-------------------------------------------------------------------------

IMPORTANT INSTALLATION NOTE:

Some Windows installations may not initially permit you to open the 
Help File/User's Guide from within LTVT due to security issues with
the "unknown publisher".

The help file is a kind of archived mini-website, and can be opened
completely separate from the main LTVT program.

If you are unable to open the help window from within LTVT try the
following:

1. Locate the help file (LTVT_UserGuide.chm -- the icon is usually a
   notebook page with a yellow question mark) and double click on it.
2. Instead of opening directly, you may see a message box with the
   Windows security shield in the lower left and the banner at the
   top reading something like "Open File - Security Warning".
3. To avoid this security check in the future, UNCHECK the box that
   says "Always ask before opening this file".
4. Click the OPEN button.  This will both open the help file
   (outside LTVT) and mark it as a file from a trusted publisher.

After following this procedure, or one similar to it, you should be 
open the help file from within LTVT.  The content will be no different
from what you see in the stand-alone form. The only real difference is
that from within LTVT you can go directly to the instructions regarding
a particular button by tapping the F1 function key when that button is
highlighted.

[Thanks to Russ Urry for providing a screenshot of the problem and
 explaining the procedure required!]

-------------------------------------------------------------------------

The files in this folder are:

1. ReadMe.txt : the present file.

2. LTVT_vN_N.exe : the executable program file for Version N.N.

3. LTVT_UserGuide.chm : the User's Guide in Microsoft Compiled HTML Help format. 
   The revision date and program version to which it applies will be indicated 
   on the title page: "Welcome to LTVT: The User's Guide".

4. lores.jpg : a low resolution version of the famous USGS shaded relief 
   texture map of the Moon (used by the program for producing the simulated 
   views).

5. Named_Lunar_Features.csv : a list of all IAU sanctioned names, used for 
   plotting and identifying the named lunar features.

6. TAI_Offset_Data.txt : the small offsets nececessary to correct Universal 
   Time to Ephemeris Time when an estimate of the lunar geometry is requested 
   for look-up in a JPL ephemeris file.

7. unxp2000.405 : a sample JPL ephemeris file for the years 2000-2050. 

8. Observatory_List.txt : a sample text file holding site locations for
   display in the "Change Location" drop-down box in the LTVT main window.
   Edit with any text processor to add/modify site list.

9. PhotoSessions.csv : a sample listing of photo dates and times to demonstrate
   LTVT's ability to locate frames potentially showing a particular feature 
   under a particular lighting.  This list is for the plates in the Consolidated
   Lunar Atlas. Note: the images listed in this file were taken in the 1960's
   and cannot be searched without adding the JPL file unxp1950.405 (covering the
   years 1950-2000).
 
You can start the program by double-clicking on LTVT.exe, but it is best to 
read the help file first. 

-------------------------------------------------------------------------

To draw your first map click on the TEXTURE button before doing anything else.  
Then try OVERLAY DOTS.  Try experimenting with different values in the 
SUB-OBSERVER and SUB-SOLAR longitude and latitude boxes.  Clicking TEXTURE 
with different values will create different views of the Moon.

You can also change the centering of the map by clicking on it.

The included sample JPL ephemeris file will allow you to automatically estimate 
the viewing/lighting geometry from anywhere on Earth for the years 2000-2050.  
For years outside that range you will need to download additional JPL files.  
The program will prompt you if it needs one. They are freely available on the 
JPL public website at:

ftp://ssd.jpl.nasa.gov/pub/eph/export/unix/ 

LTVT *cannot* compute lunar geometries without these.  When LTVT asks you 
for one of these files simply go to the site indicated above and click on 
the file you need.  Then download it to the LTVT folder.  The JPL files are 
about 4.7 MB in size for each 50 year interval.

-------------------------------------------------------------------------

For full functionality you will probably want to use texture reference maps 
with resolution higher than the sample file provided. 

A good source for the High Resolution USGS and Clementine texture files in 
simple cylindrical projection is the USGS "MAP-A-PLANET":

http://pdsmaps.wr.usgs.gov

or 

http://www.mapaplanet.org/explorer/moon.html

Consult the program help file or visit the download pages at 

  http://www.HenriksUCLA.dk 

or

  http://ltvt.wikispaces.com/

for further directions

Alternatively, your VMA (Virtual Moon Atlas) installation, if you have one, 
contains similar textures. They will be in a folder called "Textures".  
A likely path to your VMA texture files is:

C:\Program Files\VirtualMoon\Textures

The files needed are called "hires.jpg" and "hires-clem.jpg".  You can copy 
them to your LTVT folder.  Alternatively, you can browse to the existing 
copies in the file opening dialog.  If you do so, be sure to go to 
File...Save Options... so that LTVT can remember the path in the future.

For further details on how to do this, double-click on LTVT_UserGuide.chm 
and look under Getting Started... Texture Files & JPL Ephemeris Files. 
It should be possible to open the User's Guide independently of the program 
itself. The User's Guide can also be accessed from within LTVT when it is 
running. You will find direct access to these topics under the 
Help... menu item at the top of the screen.

For more recent instructions and suggestions, see the download and help 
pages at:


-------------------------------------------------------------------------

LTVT is optimized for 1024x768 displays.
The program window cannot be resized.
The display windows look best with the "Windows Classic" Desktop Theme. 
It runs only on Windows PC's.

-------------------------------------------------------------------------

Please send your comments and questions to:  jimmosher@yahoo.com

-------------------------------------------------------------------------
