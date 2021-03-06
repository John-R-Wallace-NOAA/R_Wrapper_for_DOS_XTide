Example of adding tide height data to survey data using an R wrapper around a 32-bit DOS version of XTide: 
- https://flaterco.com/xtide/xtide.html 

On Win10 - 64-bit:

Set up a data frame in R with tide station names ( https://flaterco.com/xtide/locations.html ) and time of day.

          # My specific instuctions 
          load("W:/ALL_USR/JRW/Hook & Line Survey/2019/R/Grand.2019.RData")  # Main survey data by individual hook 
          Site.Tide.Stn <- JRWToolBox::xlsxToR("W:/ALL_USR/JRW/Hook & Line Survey/2019/R on Oracle VM VirtualBox/Tide Stations.xlsx", 3)
          
          Grand.Drop <- Grand.2019[!duplicated(paste(Grand.2019$Set.ID, Grand.2019$DROPNUM)), c("Set.ID", "SITENAME", "VESNAME", "YEAR", "DATE", "DROPTIME") ]
          Grand.Drop <- JRWToolBox::match.f(Grand.Drop, Site.Tide.Stn, 'SITENAME', 'SiteName', c("XTideStationID", "XTideStationName"))
          
          # The 32-bit R ver 2.12.2 can't handle the current .RData save() binary format (going from ver 2.12.2 to 3.6.2 does work).
          write.csv(Grand.Drop, file = "Grand.Drop.csv", row.names = FALSE)   


It appears that tide station name string can be no longer than 46 characters given, perhaps, the other characters on the XTide DOS command line.
 
          # More specific instuctions 
          # In the csv file, all 'California' extensions in the site names in Grand.Drop.csv were removed
          # These three names still needed to shorten to 46 characters or less:
            # Newport Beach, Newport Bay entrance, Corona del Mar => 'Newport Beach, Newport Bay entrance, Corona'
            # 'Rincon Island, Mussel Shoals, Santa Barbara Channel' => 'Rincon Island, Mussel Shoals, Santa Barbara Ch'
            # 'Santa Monica, Municipal Pier, San Pedro Ch' => 'Santa Monica, Municipal Pier, San Pedro Ch'
          
          # Re-read in the csv and save() for a backup on Win10 - 64bit
          Grand.Drop <- read.csv("Grand.Drop.csv", head = T, stringsAsFactors = FALSE)
          save(Grand.Drop, file = 'Grand.Drop Short Site Names 5 Feb 2020.RData')
          # load("Grand.Drop Short Site Names 5 Feb 2020.RData")




On WinXP - 32 bit: 

Install DOS XTide from: https://flaterco.com/files/xtide/xtide-2.15-DOS.zip onto Oracle VM VirtualBox WinXP running under Win10.
- Read the README.DOS file - make note of "If using a DOS box (CMD.EXE) in a 32-bit version of Windows, LFN and DPMI should just work.".  
- Setting the environmental variables didn't seem to work, but XTide worked without them for results for my local area.
- I ignored the README.DOS instructions to use unzip32.exe and just used Win10's Windows Explorer (right-click on a .zip file) and then moved the files into WinXP.
- There are very helpful settings in Oracle VM VirtualBox to get copy and paste and drag and drop to work across the Windows barrier.

Download: https://flaterco.com/files/xtide/harmonics-dwf-20191229-free.tar.xz; rename harmonics-dwf-20191229-free.tcd to harmonics.tcd and put it in the main XTide directory.
- I used BreeZip for unzipping the tar ball - it took 2 steps (not sure if BreeZip comes with Win10).

After unzipping tzd2013d.zip (provided) for the 'zoneinfo' it wasn't clear if the entire 'zoneinfo' directory could just be put into the main XTide directory. 
- Very strangely, this worked for the first time I opened a DOS Window, but not after that. Putting the 'America' directory into the main XTide directory solved this problem (wasted at least an hour on that issue).

Test XTide with:

     shell('tide -l "Wilson Cove, San Clemente Island, California" -b "2018-09-01 00:00"')

    
Move Grand.Drop.csv to WinXP and read into R ver 2.12.2: https://cran-archive.r-project.org/bin/windows/base/old/2.12.2/

Run the following code, note that the DOS XTide appends to the output file each time it is run, so the output file needs to be deleted each time after being read into R.


Helper function below is from: https://github.com/John-R-Wallace-NOAA/JRWToolBox 
- base::strsplit() could be used, but it is a little more awkward.
       
       get.subs <- function (x, sep = ",", collapse = F)       {
          subs <- function(x, sep = ",") {
              " #   DATE WRITTEN:  26 May 1995,  Revised Apr 2017  "
              " #   Author:  John R. Wallace (John.Wallace@noaa.gov)  "
              "  "
              if (length(sep) == 1) 
                  sep <- substring(sep, 1:nchar(sep), 1:nchar(sep))
              nc <- nchar(x)
              y <- (1:nc)[is.element(substring(x, 1:nc, 1:nc), sep)]
              if (is.na(y[1] + 0)) 
                  return(x)
              substring(x, c(1, y + 1), c(y - 1, nc))
          }        
          "   "
          if (length(x) == 1) {
              if (collapse) 
                  paste(subs(x, sep = sep), collapse = "")
              else subs(x, sep = sep)
          }
          else {
              if (collapse) 
                  apply(matrix(x, ncol = 1), 1, function(x) paste(subs(x, 
                      sep = sep), collapse = ""))
              else apply(matrix(x, ncol = 1), 1, subs, sep = sep)
          }
      }
 
 
Read in the csv file and fill the XTide.m column with NA's:

    Grand.Drop <- read.csv("Grand.Drop.csv", head = T, stringsAsFactors = FALSE)
    Grand.Drop$XTide.m <- NA

Switching from my date format: 

            Set.ID SITENAME VESNAME YEAR       DATE DROPTIME XTideStationID          XTideStationName XTide.m
    1 04-01-01-001      205  Mirage 2004 11/10/2004     7:46        9411340 Santa Barbara, California      NA
    2 04-01-01-001      205  Mirage 2004 11/10/2004     8:15        9411340 Santa Barbara, California      NA
    3 04-01-01-001      205  Mirage 2004 11/10/2004     8:31        9411340 Santa Barbara, California      NA


To XTide's format of: "YYYY-MM-DD HH:MM" and adding one minute (60 secs) to get a range of time:
- Note that leading or following extra spaces in the character strings will break XTide.

      for ( i in 1:nrow(Grand.Drop)) {
    
        cat(i, "\n\n")
       
        Date <- paste(get.subs(Grand.Drop$DATE[i], "/")[c(3,1,2)], collapse= "-")
        DropTime <- Grand.Drop$DROPTIME[i]
       
        hourMin <- get.subs(as.character(strptime (DropTime, format = "%H:%M") + 60), sep = ":")[1:2]
        hourMin[1] <- get.subs(hourMin[1], sep = " ")[2]
        hourMin <- paste(hourMin,  collapse = ":")
       
        STRING <- paste('tide -l "', Grand.Drop$XTideStationName[i], '" -b "', Date, ' ', DropTime, '" -e "', Date, ' ',hourMin, '" -o tmp.txt -s "00:01" -m m', sep="")
        shell(STRING)
        read.table("tmp.txt", head = FALSE)
       
        Grand.Drop$XTide.m[i] <- as.numeric(unlist(as.vector(read.table("tmp.txt", head = F))))[5]
        file.remove('tmp.txt')
      }
      
   
      
Final result:
      
            Set.ID SITENAME VESNAME YEAR       DATE DROPTIME XTideStationID          XTideStationName  XTide.m
    1 04-01-01-001      205  Mirage 2004 11/10/2004     7:46        9411340 Santa Barbara, California 5.784617
    2 04-01-01-001      205  Mirage 2004 11/10/2004     8:15        9411340 Santa Barbara, California 5.605676
    3 04-01-01-001      205  Mirage 2004 11/10/2004     8:31        9411340 Santa Barbara, California 5.447639

    save(Grand.Drop, file = 'Grand.Drop, XTide values, 5 Feb 2020.RData')
      
      
      
    



