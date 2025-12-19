 # process only for SECAL samples
    if data[2] == 'cecal':
        value = math.log10(float(data[8]))
        if data[5] == 'ABX':
            cecal_abx.append(value)
        else:
            cecal_plb.append(value)
    # process only for ILEAL samples
    if data[2] == 'ileal':
        value = math.log10(float(data[8]))
        if data[5] == 'ABX':
            ileal_abx.append(value)
        else:
            ileal_plb.append(value)
    
    line = fd.readline()
fd.close()


# draw curve    
axis.violinplot([cecal_abx, cecal_plb, ileal_abx, ileal_plb])

figure.savefig("out.png", dpi=200)







for mouse_number in range(1, 17):
   if mouse_number < 10:
      mouse_id = f"ABX00{mouse_number}"
    elif mouse_number < 100:#        mouse_id = f"ABX0{mouse_number}"
    else:
        mouse_id = f"ABX{mouse_number}"
        
    x = []
    y = []
    fd = open("data_small.csv", "r")
    # skip 1st line
    line = fd.readline()
    # retrieve 1st data line (2nd line)
    line = fd.readline()
    while line != '':
        # remove endof line character
        line = line.replace("\n", "")
        data = line.split(";")
        # filter by mouse ID
        if data[4] == mouse_id:
            # get color from treatment
            if data[5] == 'ABX':
                c = 'red'
            else:
                c = 'blue'

            # process only for FECAL samples
            if data[2] == 'fecal':
                xvalue = float(data[7])
                yvalue = math.log10(float(data[8]))
                x.append(xvalue)
                y.append(yvalue)
            
        line = fd.readline()
    fd.close()
