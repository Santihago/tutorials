Example script (skipping step 1 and 2), for multiple participants:
https://www.dropbox.com/s/t2hrjp9qwe5xrlx/time_statplot_group.m?dl=0

# Between-trials cluster-based permutation test

## 1. Selecting trials

We will first select the trials for our planned comparison. Since we plan to contrast two conditions, we select trials using the event codes corresponding to the two conditions. This step can be carried out both in EEGLAB and Fieldtrip.

If the data is already epoched and in EEGLAB's format (`.set`), a fast way is to select trials directly on EEGLAB. Using EEGLAB’s visual interface, go to `‘Edit’ > ‘Select epochs or events’` and then manually selecting the events in the list.

## 2. Convert the datafiles to Fieldtrip’s format. 

EEGLAB data is visible in the variable workspace under the name ‘EEG’. This data structure contains all the EEG data for a single .set file. However, Fieldtrip data is structured differently than Eeglab data. Luckily there is a function that transforms data from one format to the other. Use the following code to convert one dataset:

```Matlab
data_all = eeglab2fieldtrip(EEG,'raw')
```

If you click on the newly created ‘data_all’ and explore the data structure, you will observe that trial information is now located in the trialinfo subfield.

## 3. Now we need to separate our data into two different data structures, one for each condition. This step is necessary for the permutation test. For this step, we will use a Fieldtrip's function to select the desired trials.

Using Fieldtrip code:
Check the data_all.trialinfo to see what your event codes are. If they are in text format, you can use e.g. strcmp(data_all.trialinfo.type, ‘SNTD’);. Here we have two event types, 72 and 73.
```Matlab
cfg = [];
cfg.trials = data_all.trialinfo == 72;
data_A = ft_redefinetrial(cfg, data_all);

cfg = [];
cfg.trials = data_all.trialinfo == 73;
data_B = ft_redefinetrial(cfg, data_all);
```

This will create two data structures, data_A and data_B (containing the trials for each of the conditions).

## 4. For each of the condition's data structures, we will now average across all trials using ft_timelockanalysis. We do not average across electrodes or time.

Note: For analysis of a single participant, we want cfg.keeptrials = 'yes'. That is because we will compare multiple trials of one condition with the trials in another condition. For multiple participants, Fieldtrip can’t deal with both multiple trials per participant and multiple participants. So we will use averages per condition, akin to a Repeated Measures where whe average across trials per condition for each participant. In that case, we use cfg.keeptrials = 'no'. This will affect the structure of the resulting timelock data and how we define the Design Matrix later.
Note 2: For a dependent samples t-test, the number of trials should match between conditions.

% We average all trials per condition 
cfg = [];
cfg.keeptrials = 'yes';
timelock_A    = ft_timelockanalysis(cfg, data_A);
timelock_B    = ft_timelockanalysis(cfg, data_B);

## 5. We will prepare an electrode layout to calculate clusters (a map indicating which electrodes are close to each other) , which will depend on our specific setup. 

Luckily we have a layout for our Geodesics electrode cap in Fieldtrip's documents. This will also be used later for our topographical plots.

```Matlab
%% Prepare the EEG layout for plotting

[~,ftpath] = ft_version;
elec = ft_read_sens(strcat(ftpath, '/template/electrode/GSN-HydroCel-129.sfp' ));

% Create a template with electrode neighbours
cfg = [];
cfg.method = 'triangulation';
cfg.neighbourdist = 4;
cfg.feedback = 'no' ;
neighbours = ft_prepare_neighbours(cfg, elec);
```

## 6.  We create a design matrix. 
This step can be confusing at first. The Design Matrix will be different depending on whether we are doing a within-subjects or between-subjects comparison. Here is where we indicate whether we have multiple participants with multiple conditions each, or just one participant, or multiple participants who did different conditions, and so on. For instance, for two participants who completed two conditions, the matrix would be:
1 2 1 2 : ivar
1 1 2 2 : uvar

Where the first row (ivar) indicates the independent variable (condition 1 or 2), and the second row (uvar) indicates the unit variable (subject 1 or 2).
We can semi automatise its creation if we previously define the number of subjects and of conditions for the experiment, as follows:

```Matlab
conditions = [1 2];
cfg.design(1,:) = repmat(conditions, 1, length(subjects));  % IV
cfg.design(2,:) = repelem(1:length(subjects), length(conditions));
cfg.ivar = 1;  %  1st row of the design matrix is the ivar
cfg.uvar = 2;  % 2nd row of the design matrix is the uvar
```

For a single participant, where we are not using averages per condition, the design matrix is a bit different. Each data structure (A and B) will contain data from multiple trials. Se we need to tell Fieldtrip to what condition each trial corresponds to.
For example, as done in the official tutorial:

```Matlab
n_fc  = size(timelockFC.trial, 1);
n_fic = size(timelockFIC.trial, 1);

cfg.design           = [ones(1,n_fic), ones(1,n_fc)*2]; % design matrix
cfg.ivar             = 1; % number or list with indices indicating the independent variable(s)
```

## 7. We submit the two data structures and the design matrix to Fieldtrip's permutation test. 

Since we are dealing with two conditions from the same participant, we use the dependent samples t-test (ft_statfun_depsamplesT). We leave a

```Matlab
%% PERMUTATION TEST

cfg = [];

cfg.latency          = [0  Inf];  % Inf is the last value in our epoch
cfg.method           = 'montecarlo';
cfg.statistic        = 'ft_statfun_depsamplesT';  % Alternatives: 'indepsamplesT'; 'ft_statfun_depsamplesregrT';
% calculates independent samples regression coefficient
% t-statistics on the biological data (the dependent variable), using the information
% on the independent variable (predictor) in the design.
cfg.numrandomization = 1000;
cfg.correctm         = 'cluster';
cfg.clusteralpha     = 0.05;
cfg.clusterstatistic = 'maxsum';

cfg.elec             = 'all';
%cfg.neighbourdist    = 4;
cfg.neighbours       = neighbours;
cfg.computecritval   = 'yes';
cfg.computeprob      = 'yes';
%cfg.tail             = 1;          % -1, 1 or 0 (default = 0); one-sided or two-sided test
%cfg.clustertail      = 1;
cfg.alpha            = 0.05;      % alpha level of the permutation test

% DESIGN MATRIX
conditions = [1 2];
cfg.design(1,:) = repmat(conditions, 1, length(subjects));  % IV
cfg.design(2,:) = repelem(1:length(subjects), length(conditions));
cfg.ivar = 1;  % 1st row of the design matrix
cfg.uvar = 2;  % 2nd row of the design matrix

cfg.parameter = 'avg';
stat = ft_timelockstatistics(cfg, ...
     timelock_A{:}, ...
     timelock_B{:});
```

## 8. Create a plot of the t-values per electrode

```Matlab
%% PLOT

% Plot the t-values
cfg = [];
cfg.elec = elec;
cfg.parameter = 'stat'; 
figure; ft_topoplotER(cfg,stat); colorbar
%

% Plot
cfg = [];
%cfg.zlim  = [-6 6]; % T-values
%cfg.alpha = 0.05;
%cfg.elec  = elec;
ft_clusterplot(cfg, stat);

% so far it was the same as above, now change the colormap
ft_hastoolbox('brewermap', 1);         % ensure this toolbox is on the path
colormap(flipud(brewermap(64,'RdBu'))) % change the colormap
```


This is an example resulting figure:





## 9. Interpreting the stats

Statistical results are now inside the ‘stat’ data structure. Click to explore:

```
prob: [91×151 double]
posclusters: [1×4 struct]
posclusterslabelmat: [91×151 double]
posdistribution: [1×1000 double]
negclusters: [1×2 struct]
negclusterslabelmat: [91×151 double]
negdistribution: [1×1000 double]
cirange: [91×151 double]
mask: [91×151 logical]
stat: [91×151 double]
ref: [91×151 double]
critval: [-2.0860 2.0860]
df: 20
dimord: 'chan_time'
elec: [1×1 struct]
label: {91×1 cell}
time: [1×151 double]
cfg: [1×1 struct]
```



Clusters will be found in posclusters or negclusters. They will contain the cluster p-value (prob), the cluster statistic with its standard deviation and confidence interval.
