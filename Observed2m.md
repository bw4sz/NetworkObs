# Trait-matching and Available Resources
Ben Weinstein - Stony Brook University  




```
## [1] "Run Completed at 2016-06-06 16:33:07"
```


```r
#reload if needed
#load("Observed.Rdata")
```

#Load in data


```r
#read in flower morphology data, comes from Nectar.R
droppath<-"C:/Users/Ben/Dropbox/"
fl.morph<-read.csv(paste(droppath,"Thesis/Maquipucuna_SantaLucia/Results/FlowerMorphology.csv",sep=""))

#First row is empty
fl.morph<-fl.morph[-1,]

#Bring in Hummingbird Morphology Dataset, comes from
hum.morph<-read.csv(paste(droppath,"Thesis/Maquipucuna_SantaLucia/Results/HummingbirdMorphology.csv",sep=""))

#taxonomy change, we are calling them Crowned Woodnymph's now.
hum.morph$English<-as.character(hum.morph$English)

hum.morph$English[hum.morph$English %in% "Green-crowned Woodnymph"]<-"Crowned Woodnymph"

#Bring in Interaction Matrix
int<-read.csv(paste(droppath,"Thesis/Maquipucuna_SantaLucia/Results/Network/HummingbirdInteractions.csv",sep=""),row.names=1)

#one date error
int[int$DateP %in% '2013-07-25',"Month"]<-7

#one duplicate camera error, perhaps two GPS records.
int<-int[!(int$ID %in% "FH1108" & int$Date_F %in% '2014-12-01'),]

#Correct known taxonomic disagreements, atleast compared to traits
int[int$Iplant_Double=="Alloplectus purpureus","Iplant_Double"]<-"Glossoloma purpureum"
int[int$Iplant_Double=="Capanea affinis","Iplant_Double"]<-"Kohleria affinis"
int[int$Iplant_Double=="Columnea cinerea","Iplant_Double"]<-"Columnea mastersonii"
int[int$Iplant_Double=="Alloplectus teuscheri","Iplant_Double"]<-"Drymonia teuscheri"
int[int$Iplant_Double=="Drymonia collegarum","Iplant_Double"]<-"Alloplectus tetragonoides"

#Some reasonable level of presences, 25 points
keep<-names(which(table(int$Hummingbird) > 25))

int<-int[int$Hummingbird %in% keep & !int$Hummingbird %in% c("Sparkling Violetear"),]

m.dat<-droplevels(int[colnames(int) %in% c("ID","Video","Time","Hummingbird","Sex","TransectID","Transect_R","Iplant_Double","Pierce","DateP","Month","ele","Type")])

#Does the data come from camera or transect?
m.dat$Type<-(is.na(m.dat$TransectID))*1

m.dat$Year<-years(as.Date(m.dat$DateP))
#one missing date
m.dat$Year[m.dat$Year %in% 2012]<-2013

#Number of bird species
h_species<-nlevels(m.dat$Hummingbird)

#Number of plant species
plant_species<-nlevels(m.dat$Iplant_Double)

#Get english name
dath<-merge(m.dat,hum.morph, by.x="Hummingbird",by.y="English",keep=all)

#Merge to flowers
int.FLlevels<-levels(factor(dath$Iplant_Double))

#Which flowers are we missing info for?
missingTraits<-int.FLlevels[!int.FLlevels %in% fl.morph$X]

#print(paste("Missing Trait Information:",missingTraits))
dath<-merge(dath,fl.morph, by.x="Iplant_Double",by.y="X")

#Drop piercing events, since they don't represent correlation
#dath<-dath[!dath$Pierce %in% c("y","Y"),]
```

##Match Species to Morphology


```r
#observed traitmatching
traitmatchF<-abs(t(sapply(hum.morph$Bill,function(x){x-fl.morph$TotalCorolla})))

rownames(traitmatchF)<-hum.morph$English
colnames(traitmatchF)<-fl.morph$Group.1
```


```r
#match names #Round to 2 decimals #Convert to cm for winbugs, avoids numerical underflow
traitmatchT<-round(traitmatchF[rownames(traitmatchF) %in% dath$Hummingbird,colnames(traitmatchF) %in% dath$Iplant_Double],2)/10
traitmatchT<-traitmatchT[sort(rownames(traitmatchT)),sort(colnames(traitmatchT))]
```

##Elevation ranges

Create a binary variable whether each observation was in a low elevation or high elevation transect. We have some species that just occur at the top of the gradient, and are not present in the sampling window of flowers at the low elevation.

Accounting for non-availability.
We have to figure out which plants were sampled in which periods, and if it was sampled, the non-detection are 0 if it wasn't the non-detection are NA. then remove all the Na's.


```r
elevH<-read.csv("InputData/HummingbirdElevation.csv",row.names=1)
head(elevH)
```

```
##                 Hummingbird  Low        m   High Index
## 1            Andean Emerald 1378 1378.632 1380.0     1
## 2    White-whiskered Hermit 1340 1437.024 1614.2     1
## 3    Stripe-throated Hermit 1360 1455.084 1527.4     1
## 4         Crowned Woodnymph 1360 1523.420 2049.0     1
## 5 Rufous-tailed Hummingbird 1370 1531.929 1862.0     3
## 6  Wedge-billed Hummingbird 1331 1624.850 2003.0     3
```

```r
colnames(elevH)[5]<-"Elevation"
elevH$Bird<-1:nrow(elevH)

#high elevation or low elevation
elevP<-read.csv("InputData/PlantElevation.csv",row.names=1)
colnames(elevP)[5]<-"Elevation"
elevP$Plant<-1:nrow(elevP)
elevP$Iplant_Double<-as.character(elevP$Iplant_Double)

#Correct known taxonomic errors
elevP[elevP$Iplant_Double %in% "Alloplectus purpureus","Iplant_Double"]<-"Glossoloma purpureum"
elevP[elevP$Iplant_Double %in% "Capanea affinis","Iplant_Double"]<-"Kohleria affinis"
elevP[elevP$Iplant_Double %in% "Alloplectus teuscheri","Iplant_Double"]<-"Drymonia teuscheri"
elevP[elevP$Iplant_Double %in% "Columnea cinerea","Iplant_Double"]<-"Columnea mastersonii"
elevP[elevP$Iplant_Double %in% "Alloplectus tenuis","Iplant_Double"]<-"Drymonia tenuis"

#Merge to observed Data
#plants
dathp<-merge(dath,elevP,by="Iplant_Double")

#birds
datph<-merge(dathp,elevH,by="Hummingbird")
```

What elevation transect is each observation in?
The camera data need to be inferred from the GPS point.


```r
#cut working best on data.frame
datph<-as.data.frame(datph)

#which elevation bin is each observation within
labs<-paste(seq(1300,2500,200),seq(1500,2700,200),sep="_")

#for the couple points that have 1290 elevation, round up to 300 for convienance
datph$ele[datph$ele < 1300]<-1301
datph$Transect_R[is.na(datph$Transect_R)]<-as.character(cut(datph[is.na(datph$Transect_R),]$ele,seq(1300,2700,200),labels=labs))

#Elev for the transects is the midpoint
tran_elev<-datph[datph$Survey_Type=='Transect',"Transect_R"]
datph[datph$Survey_Type=='Transect',"ele"]<-sapply(tran_elev,function(x){
  mean(as.numeric(str_split(x,"_")[[1]]))
})
```

### Define Camera Events


```r
#ID for NA is holger transects, make the id's 1:n for each day of transect at each elevation, assuming no elevation was split across days.
datph$ID<-as.character(datph$ID)

noid<-datph[is.na(datph$ID),]

id_topaste<-paste(noid$Month,noid$Year,"Transect",sep="_")
datph[which(is.na(datph$ID)),"ID"]<-id_topaste

#Create year month combination
datph$Time<-paste(datph$Month,datph$Year,sep="_")

#Label survey type
datph$Survey_Type<-NA

mt<-!is.na(datph$TransectID)*1
datph$Survey_Type[mt==1]<-"Transect"
datph$Survey_Type[!datph$Survey_Type %in% "Transect"]<-"Camera"

#Day level
#add day ID
sdat<-split(datph,list(datph$ID),drop = T)

sdat<-lapply(sdat,function(x){
  x<-droplevels(x)
  x$Day<-as.numeric(as.factor(x$DateP))
  return(x)
})

indatraw<-rbind_all(sdat)

#Species names
for (x in 1:nrow(indatraw)){
  indatraw$Hummingbird[x]<-as.character(elevH[elevH$Bird %in% indatraw$Bird[x],"Hummingbird"])
  indatraw$Iplant_Double[x]<-as.character(elevP[elevP$Plant %in% indatraw$Plant[x],"Iplant_Double"])
}

#match the traits
traitmelt<-melt(traitmatchT)
colnames(traitmelt)<-c("Hummingbird","Iplant_Double","Traitmatch")
#dummy presence variable
indatraw$Yobs<-1

#prune columsn to make more readable
indatraw<-indatraw[,c("Hummingbird","Iplant_Double","ID","Time","Month","Year","Transect_R","ele","DateP","Yobs","Day","Survey_Type","Pierce")]
```

##Absences - accounting for non-detection

We have more information than just the presences, given species elevation ranges, we have absences as well. Absences are birds that occur at the elevation of the plant sample, but were not recorded feeding on the flower.


```r
#Only non-detections are real 0's, the rest are NA's and are removed.
#Plants not surveyed in that time period
#Hummingbirds not present at that elevation

#For each ID
ID<-unique(indatraw$ID)

#absences data frame
absences<-list()

for(t in ID){
  
  #Which plants were sampled
  a<-indatraw %>% filter(ID==t)
  
  #For each sampled transect
  trans<-unique(a$Transect_R)
  
  for(transect in trans){

    #for each date 
    datecam<-a %>% filter(Transect_R %in% transect) %>% distinct(DateP) %>% .$DateP
    
    for(Date in datecam){
      
    #for each plant along that transect at that date
    pres<-a %>% filter(Transect_R %in% transect,DateP %in% Date) %>% distinct(Iplant_Double) %>% .$Iplant_Double
    
        #Which day in samplign
        dday<-a %>% filter(Transect_R %in% transect,DateP %in% Date) %>% distinct(Day) %>% .$Day

      for (plant in pres){
        #Get mean elevation of that plant record
        camelev<- a %>% filter(Transect_R %in% transect,DateP %in% Date,Iplant_Double %in% plant) %>% .$ele %>% mean()
        
        #Which birds are present at that observation
        predh<-elevH[((elevH$Low < camelev) & (camelev < elevH$High)),"Hummingbird"]
        
        if(length(predh)==0){next}
        
        #Make absences from those )(cat not the best)
        add_absences<-data.frame(Hummingbird=predh,Iplant_Double=plant,ID=t,DateP=Date,Month=min(a$Month),Year=unique(a$Year),Transect_R=transect,ele=camelev,Day=dday,Survey_Type=unique(a$Survey_Type),Yobs=0)
        absences<-append(absences,list(add_absences))
      }
    }
  }
}
    
indatab<-rbind_all(absences)

#merge with original data
indat<-rbind_all(list(indatraw,indatab))
```


```r
#Get trait information
#match the traits
indat<-merge(indat,traitmelt,by=c("Hummingbird","Iplant_Double"))
```

Reformat index for jags.
Jags needs a vector of input species 1:n with no breaks.


```r
#Easiest to work with jags as numeric ordinal values
indat$Hummingbird<-as.factor(indat$Hummingbird)
indat$Iplant_Double<-as.factor(indat$Iplant_Double)
indat$jBird<-as.numeric(indat$Hummingbird)
indat$jPlant<-as.numeric(indat$Iplant_Double)

jagsIndexBird<-data.frame(Hummingbird=levels(indat$Hummingbird),jBird=1:length(levels(indat$Hummingbird)))
 
jagsIndexPlants<-data.frame(Iplant_Double=levels(indat$Iplant_Double),jPlant=1:length(levels(indat$Iplant_Double)))

#Similiarly, the trait matrix needs to reflect this indexing.
jTraitmatch<-traitmatchT[rownames(traitmatchT) %in% unique(indat$Hummingbird),colnames(traitmatchT) %in% unique(indat$Iplant_Double)]
```

#Resources at each month

In our model the covariate is indexed at the scale at which the latent count is considered fixed. This means we need the resource availability per month across the entire elevation gradient for each point.


```r
#Get flower transect data
full.fl<-read.csv("C:/Users/Ben/Dropbox/Thesis/Maquipucuna_SantaLucia/Results/FlowerTransects/FlowerTransectClean.csv")[,-1]

 #month should be capital 
colnames(full.fl)[colnames(full.fl) %in% "month"]<-"Month"

#group by month and replicate, remove date errors by making a max of 10 flowers, couple times where the gps places it in wrong transect by 1 to 2 meters. 
flower.month<-group_by(full.fl,Month,Year,Transect_R,Date_F) %>% dplyr::summarise(Flowers=sum(Total_Flowers,na.rm=TRUE))  %>% filter(Flowers>20)
  
#Make month abbreviation column, with the right order
flower.month$Month.a<-factor(month.abb[flower.month$Month],month.abb[c(1:12)])

#Make year factor column
flower.month$Year<-as.factor(flower.month$Year)

#get quantile for each transect
thresh<-melt(group_by(flower.month,Transect_R) %>% summarize(Threshold=quantile(log(Flowers),0.75)) )
flower.month<-merge(flower.month,thresh)
flower.month$High<-log(flower.month$Flowers)>flower.month$value

#fix the levels
levels(flower.month$Transect_R)<-c("1300m - 1500m", "1500m - 1700m","1700m - 1900m","1900m - 2100m","2100m - 2300m","2300m - 2500m")
#plot
ggplot(flower.month,aes(x=Month.a,log(Flowers),col=High,shape=as.factor(Year))) + geom_point(size=3) + theme_bw()  + geom_smooth(aes(group=1)) + ylab("Flowers") + xlab("Month") + facet_wrap(~Transect_R,scales="free_y") + labs(shape="Year", y= "Log Available Flowers") + scale_x_discrete(breaks=month.abb[seq(1,12,2)]) + scale_color_manual(labels=c("Low","High"),values=c("black","red")) + labs(col="Resource Availability")
```

<img src="figureObserved/unnamed-chunk-13-1.png" title="" alt="" style="display: block; margin: auto;" />

```r
#turn min and max elvation into seperate columns for the range
flower.month$minElev<-as.numeric(str_extract(flower.month$Transect_R,"(\\d+)"))
flower.month$maxElev<-as.numeric(str_match(flower.month$Transect_R,"(\\d+)_(\\d+)")[,3])
```


```r
indat$All_Flowers<-NA
indat$Used_Flowers<-NA
indat$FlowerA<-NA

#Resource list for each species.
slist<-int %>% group_by(Hummingbird,Iplant_Double) %>% distinct() %>% dplyr::select(Hummingbird,Iplant_Double) %>% arrange(Hummingbird)

#Create time ID for flower transects
full.fl$Time<-paste(full.fl$Month,full.fl$Year,sep="_")

for (x in 1:nrow(indat)){

 #Flowers used by the hum species
 sp_list<-slist[slist$Hummingbird %in% indat$Hummingbird[x],"Iplant_Double"]
  
 #Species of flower
 sp <-  indat[x,"Iplant_Double"]
 
 #Filter by Date
 bydate<-full.fl[full.fl$Month %in% indat$Month[x] & full.fl$Year %in% indat$Year[x],]
 
 #average number of flowers at each elevation within that time frame.
 
 #count number of all flowers
 indat$All_Flowers[x]<- bydate%>% group_by(Date_F,Transect_R) %>% summarize(n=sum(Total_Flowers,na.rm=T)) %>% group_by(Transect_R) %>% summarize(mn=mean(n)) %>% summarize(F=sum(mn)) %>% .$F
 
 #filter by species
  byspecies<-bydate[bydate$Iplant_Double %in% sp_list$Iplant_Double,]
 
  indat$Used_Flowers[x]<-byspecies%>% group_by(Date_F,Transect_R) %>% summarize(n=sum(Total_Flowers,na.rm=T)) %>% group_by(Transect_R) %>% summarize(mn=mean(n)) %>% summarize(F=sum(mn)) %>% .$F
  
  #just the abundance of that species
  bys <-  bydate[bydate$Iplant_Double %in% sp,]
  indat$FlowerA[x] <-  bys %>% group_by(Date_F,Transect_R) %>% summarize(n=sum(Total_Flowers,na.rm=T)) %>% group_by(Transect_R) %>% summarize(mn=mean(n)) %>% summarize(F=sum(mn)) %>% .$F

}
```

###Relationship between resource measures


```r
ggplot(indat,aes(x=All_Flowers,y=Used_Flowers)) + geom_point() + facet_wrap(~Hummingbird,scales="free")
```

<img src="figureObserved/unnamed-chunk-15-1.png" title="" alt="" style="display: block; margin: auto;" />

##Binary Measures of Resources


```r
#All Resources
indat$BAll_Flowers<-(indat$All_Flowers > quantile(indat$All_Flowers,0.75))*1
#mnth<-sapply(indat$Time,function(x){
 # as.numeric(str_split(x,"_")[[1]][1])})
#indat$BAll_Flowers<-(mnth  %in% c(6,7,8,9,10))*1

qthresh<-indat %>% group_by(Hummingbird) %>% summarize(UThresh=mean(Used_Flowers))

#save a copy in case you need it here
tosave<-indat

indat<-merge(indat,qthresh)
indat$BUsed_Flowers<-(indat$Used_Flowers > indat$UThresh)*1

fthresh<-indat %>% group_by(Hummingbird) %>% summarize(FThresh=mean(FlowerA))
indat<-merge(indat,fthresh)
indat$BFlowerA<-(indat$FlowerA > indat$FThresh)*1
```


```r
#Combine resources with observed data
#As matrix
indat$Transect<-(indat$Survey_Type=="Transect")*1
indat$Camera<-(indat$Survey_Type=="Camera")*1

indat<-droplevels(indat)

#Turn Time and ID into numeric indexes
indat$jTime<-as.numeric(as.factor(indat$Time))
indat$jID<-as.numeric(as.factor(indat$ID))

#index resources
resourceMatrix<-indat %>% group_by(jBird,jPlant,jID) %>% summarize(v=length(mean(Used_Flowers))) %>% mutate(v=scale(v)) %>% acast(jBird ~ jPlant ~ jID,value.var='v',fill=0)

#bind to indat for later
mr<-melt(resourceMatrix)
colnames(mr)<-c("jBird","jPlant","jTime","scaledR")
indat<-merge(indat,mr,by=c("jBird","jPlant","jTime"))

##melted version for plotting
mindat<-melt(indat,measure.vars=c("Camera","Transect"))
```

###Calculate Sampling effort


```r
#At each data point there is a camera a transect or both
cam_effort<-(!is.na(indat$Camera))*1
trans_effort<-(!is.na(indat$Transect))*1
```

##View species identity in resource splits.


```r
#Count of species in both time sets
splist<-mindat %>% filter(value>0) %>% group_by(Hummingbird,Resource= BUsed_Flowers) %>% distinct(Iplant_Double) %>% dplyr::select(Iplant_Double)

#relevel text
splist$Resource<-factor(as.factor(as.character(splist$Resource)),labels = c("Low","High"))

splist<-split(splist,splist$Hummingbird)
p<-list()
for (x in 1:length(splist)){
  split_sp<-split(splist[[x]]$Iplant_Double,splist[[x]]$Resource)
  p[[x]]<-venn.diagram(split_sp,filename=NULL,scaled=T,main =unique(splist[[x]]$Hummingbird),fill=c("Blue","Red"),alpha=c(.25,.75),cat.cex=1.5,cex=.75,main.cex=2)

  #get index
  labs<-lapply(p[[x]],function(i) i$label)
  
  in_low<-which(labs == "Low")

  in_high<-which(labs == "High")

#edit labels, depends on length
#which is low
  if(length(p[[x]])==10){
  p[[x]][[in_low-3]]$label<-paste(split_sp$Low[!split_sp$Low %in% split_sp$High],collapse="\n")

  p[[x]][[in_high-3]]$label<-paste(split_sp$High[!split_sp$High %in% split_sp$Low],collapse="\n")
  }
  
  if(length(p[[x]])==9 & !is.null(labs[[4]])){
      p[[x]][[in_low-3]]$label<-paste(split_sp$Low[!split_sp$Low %in% split_sp$High],collapse="\n")

  p[[x]][[in_high-1]]$label<-paste(split_sp$High[!split_sp$High %in% split_sp$Low],collapse="\n")
  }
  if(length(p[[x]])==9 & is.null(labs[[4]])){
      p[[x]][[in_low-2]]$label<-paste(split_sp$Low[!split_sp$Low %in% split_sp$High],collapse="\n")

  p[[x]][[in_high-2]]$label<-paste(split_sp$High[!split_sp$High %in% split_sp$Low],collapse="\n")
  }
  grid.newpage()
  grid.draw(p[[x]])
}
```

<img src="figureObserved/unnamed-chunk-19-1.png" title="" alt="" style="display: block; margin: auto;" /><img src="figureObserved/unnamed-chunk-19-2.png" title="" alt="" style="display: block; margin: auto;" /><img src="figureObserved/unnamed-chunk-19-3.png" title="" alt="" style="display: block; margin: auto;" /><img src="figureObserved/unnamed-chunk-19-4.png" title="" alt="" style="display: block; margin: auto;" /><img src="figureObserved/unnamed-chunk-19-5.png" title="" alt="" style="display: block; margin: auto;" /><img src="figureObserved/unnamed-chunk-19-6.png" title="" alt="" style="display: block; margin: auto;" /><img src="figureObserved/unnamed-chunk-19-7.png" title="" alt="" style="display: block; margin: auto;" /><img src="figureObserved/unnamed-chunk-19-8.png" title="" alt="" style="display: block; margin: auto;" /><img src="figureObserved/unnamed-chunk-19-9.png" title="" alt="" style="display: block; margin: auto;" /><img src="figureObserved/unnamed-chunk-19-10.png" title="" alt="" style="display: block; margin: auto;" /><img src="figureObserved/unnamed-chunk-19-11.png" title="" alt="" style="display: block; margin: auto;" /><img src="figureObserved/unnamed-chunk-19-12.png" title="" alt="" style="display: block; margin: auto;" /><img src="figureObserved/unnamed-chunk-19-13.png" title="" alt="" style="display: block; margin: auto;" />

```r
#venn diagram writes a silly set of log files
file.remove(list.files(pattern="*.log"))
```

```
## [1] TRUE TRUE TRUE TRUE TRUE TRUE TRUE
```



```r
gc()
```

```
##           used (Mb) gc trigger  (Mb) max used  (Mb)
## Ncells 1762045 94.2    3205452 171.2  3205452 171.2
## Vcells 5000320 38.2    8415273  64.3  8412780  64.2
```

```r
save.image("Observed.Rdata")
```

