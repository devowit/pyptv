# todo list for the multiplane calibration

introduction
multiplane calibration logic is the following: 
- create parameter sets per each plane (see `test_multiplane` or `multiplane_calibration`)
- in these sets, set "calibrate from Z" to 1 and "combine" to 0
- in each plane do calibration on a 2D target, save .crd and .fix files 
![](https://github.com/OpenPTV/openptv-python/blob/535c94676ab173b680b2b0f95a5bd9a44ae06fe7/src_c/jw_ptv.c#L752)
or see below the code 

*Note*: in each folder /cal for each plane the 'z' values in the calibration_target.txt file
are correct, this is how we know which plane is where. It also allows us to use different
point numbering and see different points. BUT, at the end we have to combine them and 
number again (providing unique id to each point). The question is whether we need to 
preserve the original numbering (as it was before we asked the users to number points in 
one plane as 101, 102,... and in another plane 201, 202, - if the number of points is less
than 100 of course. 


* todo discover what are these files and who writes them? python/liboptv/cython? 
let's check in PBI - it's just `numpy.savetxt`

```
    def save_point_sets(self, detected_file, matched_file, cal_points):
        """
        Save text files of the intermediate results of detection and of 
        matching the detections to the known positions. This may later be used
        for multiplane calibration.
        
        Currently assumes that the first targets up to the number of known 
        points are the matched points.
        
        Arguments:
        detected_file - will contain the detected points in a 3 column format -
            point number, x, y.
        matched_file - same for the known positions that were matched, with 
            columns for point number, x, y, z.
        cal_points - the known points array, as in other methods.
        """
        detected = np.empty((len(cal_points), 2))
       
        nums = np.arange(len(cal_points))
        for pnr in nums:
            detected[pnr] = self._targets[pnr].pos()
        
        detected = np.hstack((nums[:,None], detected))
        known = np.hstack((nums[:,None], cal_points))
        
        # Formats from jw_ptv.c, until we can rationalize this stuff.        
        np.savetxt(detected_file, detected, fmt="%9.5f")
        np.savetxt(matched_file, known, fmt="%10.5f")
 ```       




- create additional parameter set, called multiplane or combination
- in this set, set "calibrate from Z" to 1 and "combine" to 1
- create parameters/multi_planes.par - not yet in GUI 
* todo - think if we need it in GUI
* todo - think how to do it in GUI
- load image (whatever plane), regular procedure (detection, ...) then directly to 
Raw orientation - if both examine parameters are set to 1, then do the following: (*todo)

    - read the multi_planes.par
    - from each multi plane folder (according to the list in the par file): 
        a) read crd and fix
        b) combine this info into a 3D calibration target (virtual 3d target) and virtual points
        c) run the usual Raw Orientation and then Fine Orientation
        
    this step is identical to pbi/ptv/multiplane.py, it's just using YAML instead of PAR
    to keep the information about the files with the dots and detections. 
                
            
Under Orientation part: 


1. saving
```
            for (i_img = 0; i_img < cpar->num_cams; i_img++)
            {
                for (i=0; i<nfix ; i++)
                {
                    pixel_to_metric(&crd[i_img][i].x, &crd[i_img][i].y,
                        pix[i_img][i].x, pix[i_img][i].y, cpar);
                    crd[i_img][i].pnr = pix[i_img][i].pnr;
                }
                
                /* save data for special use of resection routine */
                if (examine == 4 && multi==0)
                {
                    printf ("try write resection data to disk\n");
                    /* point coordinates */
                    //sprintf (filename, "resect_%s.fix", img_name[i_img]);
                    sprintf (filename, "%s.fix", img_name[i_img]);
                    write_ori (Ex[i_img], I[i_img], G[i_img], ap[i_img],
                        img_ori[i_img], NULL); /* ap ignored */
                    fp1 = fopen (filename, "w");
                    for (i=0; i<nfix; i++)
                        fprintf (fp1, "%3d  %10.5f  %10.5f  %10.5f\n",
                                 fix[i].pnr, fix[i].x, fix[i].y, fix[i].z);
                    fclose (fp1);
                    
                    /* metric image coordinates */
                    //sprintf (filename, "resect_%s.crd", img_name[i_img]);
                    sprintf (filename, "%s.crd", img_name[i_img]);
                    fp1 = fopen (filename, "w");
                    for (i=0; i<nfix; i++)
                        fprintf (fp1,
                                 "%3d  %9.5f  %9.5f\n", crd[i_img][i].pnr,
                                 crd[i_img][i].x, crd[i_img][i].y);
                    fclose (fp1);
                    
                    /* orientation and calibration approx data */
                    write_ori (Ex[i_img], I[i_img], G[i_img], ap[i_img],
                        "resect.ori0", "resect.ap0");
                    printf ("resection data written to disk\n");
                }
                
 ```               
2. if it's a multi-plane folder, then "resection":

```
                /* resection with dumped datasets */
                if (examine == 4)
                {
                    printf("Resection with dumped datasets? (y/n)");
                    //scanf("%s",buf);
                    //if (buf[0] != 'y')	continue;
                    //strcpy (buf, "");
                    if (multi == 0)	continue;
                    
                    /* read calibration frame datasets */
                    //sprintf (multi_filename[0],"img/calib_a_cam");
                    //sprintf (multi_filename[1],"img/calib_b_cam");
                    
                    fp1 = fopen_r ("parameters/multi_planes.par");
                    fscanf (fp1,"%d\n", &planes);
                    for(i=0;i<planes;i++) 
                        fscanf (fp1,"%s\n", multi_filename[i]);
                        //fscanf (fp1,"%s\n", &multi_filename[i]);
                    fclose(fp1);
                    for (n=0, nfix=0, dump_for_rdb=0; n<10; n++)
                    {
                        //sprintf (filename, "resect.fix%d", n);
                        
                        sprintf (filename, "%s%d.tif.fix", multi_filename[n],i_img+1);
                        
                        fp1 = fopen (filename, "r");
                        if (! fp1)	continue;
                        
                        printf("reading file: %s\n", filename);
                        k = 0;
                        while ( fscanf (fp1, "%d %lf %lf %lf",
                                        &fix[nfix+k].pnr, &fix[nfix+k].x,
                                        &fix[nfix+k].y, &fix[nfix+k].z)
                               != EOF) k++;
                        fclose (fp1);
                                                
                        /* read metric image coordinates */
                        //sprintf (filename, "resect_%d.crd%d", i_img, n);
                        sprintf (filename, "%s%d.tif.crd", multi_filename[n],i_img+1);
                        printf("reading file: %s\n", filename);
                        fp1 = fopen (filename, "r");
                        if (! fp1)	continue;
                        
                        for (i=nfix; i<nfix+k; i++)
                            fscanf (fp1, "%d %lf %lf",
                                    &crd[i_img][i].pnr,
                                    &crd[i_img][i].x, &crd[i_img][i].y);
                        nfix += k;
                        fclose (fp1);
                    }
                    
                    printf("nfix = %d\n",nfix);
                    
                    /* resection */
                    /*Beat Mai 2007*/
                    sprintf (filename, "raw%d.ori", i_img);
                    UPDATE_CALIBRATION(i_img, &Ex[i_img], &I[i_img], &G[i_img], filename,
                        &(ap[i_img]), "addpar.raw", NULL);
                    
                    /* markus 14.05.2007 show coordinates combined */
                    for (i=0; i<nfix ; i++)			  
                    {
                        /* first crd->pix */
                        metric_to_pixel(&pix[i_img][i].x, &pix[i_img][i].y, 
                            crd[i_img][i].x, crd[i_img][i].y, cpar);
                    }
                    
                    
                    orient_v3 (Ex[i_img], I[i_img], G[i_img], ap[i_img], *(cpar->mm),
                               nfix, fix, crd[i_img],
                               &Ex[i_img], &I[i_img], &G[i_img], &ap[i_img], i_img,
                               resid_x, resid_y, pixnr,
                               orient_n + i_img);
                    for (i = 0; i < orient_n[i_img]; i++) {
                        orient_x1[i_img][i] = pix[i_img][pixnr[i]].x;
                        orient_y1[i_img][i] = pix[i_img][pixnr[i]].y;
                        orient_x2[i_img][i] = pix[i_img][pixnr[i]].x + 5000*resid_x[i];
                        orient_y2[i_img][i] = pix[i_img][pixnr[i]].y + 5000*resid_y[i];
                    }
                    ///////////////////////////////////////////
                    
                    
                }
```