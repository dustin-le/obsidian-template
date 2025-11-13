	#dailynote
	
	# <% moment(tp.file.title,'YYYY-MM-DD').format("dddd, MMMM DD, YYYY") %>
	
	---
	### 📅 Daily Work
	
	#### ⏰ <%*
	// Function to generate a date file name in the required format
	function getDateFileName(momentDate) {
	    const dayOfWeek = momentDate.format('dddd');
	    return `${momentDate.format('YYYY-MM-DD')}-${dayOfWeek}`;
	}
	
	// Function to get the folder path for a given date
	function getFolderPath(momentDate) {
	    const year = momentDate.format('YYYY');
	    const monthNum = momentDate.format('MM');
	    
	    // Month names matching your folder structure
	    const monthNames = {
	        '01': 'January',   '02': 'February',  '03': 'March',     '04': 'April',
	        '05': 'May',       '06': 'June',      '07': 'July',      '08': 'August', 
	        '09': 'September', '10': 'October',   '11': 'November',  '12': 'December'
	    };
	    
	    const monthName = monthNames[monthNum];
	    
	    // Use the detected base path from the current file location
	    const currentFilePath = tp.file.path(true);
	    const pathParts = currentFilePath.split('/');
	    let basePath = '';
	    
	    // Structure: Daily Notes/2025/09-September/filename.md
	    // We want: Daily Notes/
	    if (pathParts.length > 3) {
	        // Remove filename, month folder, and year folder - keep everything before that
	        basePath = pathParts.slice(0, -3).join('/') + '/';
	    }
	    
	    return `${basePath}${year}/${monthNum}-${monthName}`;
	}
	
	// Function to get the full file path for a given date
	function getFullFilePath(momentDate) {
	    const folderPath = getFolderPath(momentDate);
	    const fileName = getDateFileName(momentDate);
	    return `${folderPath}/${fileName}.md`;
	}
	
	// Function to extract unfinished tasks from a daily note content
	function extractUnfinishedTasks(content) {
	    const unfinishedTasks = [];
	    
	    // Find the Daily Work section - improved regex to not stop at subsections
	    const dailyWorkMatch = content.match(/### 📅 Daily Work([\s\S]*?)(?=---|\n### |\n# |$)/);
	    if (!dailyWorkMatch) return unfinishedTasks;
	    
	    const dailyWorkSection = dailyWorkMatch[1];
	    
	    // Extract all unchecked tasks from the subsections
	    const taskSections = [
	        /#### ⏰.*?\.\.\.([\s\S]*?)(?=####|###|---|\n#|$)/,
	        /#### 🚀 Things I worked on today\.\.\.([\s\S]*?)(?=####|###|---|\n#|$)/,
	        /#### ⛅ Things I need to do tomorrow\.\.\.([\s\S]*?)(?=####|###|---|\n#|$)/
	    ];
	    
	    taskSections.forEach(regex => {
	        const sectionMatch = dailyWorkSection.match(regex);
	        if (sectionMatch) {
	            const sectionContent = sectionMatch[1];
	            // Find all unchecked tasks (both - [ ] and * [ ] formats)
	            const unfinishedMatches = sectionContent.match(/^[*-] \[ \] (.+)$/gm);
	            if (unfinishedMatches) {
	                unfinishedMatches.forEach(task => {
	                    // Extract just the task text (remove the checkbox part)
	                    const taskText = task.replace(/^[*-] \[ \] /, '');
	                    if (taskText.trim()) {
	                        // Check if #carryover tag already exists
	                        if (taskText.includes('#carryover')) {
	                            unfinishedTasks.push(`- [ ] ${taskText}`);
	                        } else {
	                            unfinishedTasks.push(`- [ ] ${taskText} #carryover`);
	                        }
	                    }
	                });
	            }
	        }
	    });
	    
	    return unfinishedTasks;
	}
	
	// Function to find the most recent daily note within the current month or previous month
	function findMostRecentNote() {
	    const today = moment(tp.file.title, 'YYYY-MM-DD');
	    const firstOfMonth = today.clone().startOf('month');
	    const firstOfPreviousMonth = today.clone().subtract(1, 'month').startOf('month');
	    
	    // Start from yesterday and go backwards
	    let searchDate = today.clone().subtract(1, 'day');
	    
	    // Search within current month first
	    while (searchDate.isSameOrAfter(firstOfMonth)) {
	        const filePath = getFullFilePath(searchDate);
	        const file = app.vault.getAbstractFileByPath(filePath);
	        
	        if (file && file.path) {
	            return { 
	                file, 
	                fileName: getDateFileName(searchDate), 
	                filePath: filePath,
	                date: searchDate.clone() 
	            };
	        }
	        
	        searchDate.subtract(1, 'day');
	    }
	    
	    // If no note found in current month, search the previous month
	    // Start from the last day of previous month
	    searchDate = today.clone().subtract(1, 'month').endOf('month');
	    
	    while (searchDate.isSameOrAfter(firstOfPreviousMonth)) {
	        const filePath = getFullFilePath(searchDate);
	        const file = app.vault.getAbstractFileByPath(filePath);
	        
	        if (file && file.path) {
	            return { 
	                file, 
	                fileName: getDateFileName(searchDate), 
	                filePath: filePath,
	                date: searchDate.clone() 
	            };
	        }
	        
	        searchDate.subtract(1, 'day');
	    }
	    
	    return null;
	}
	
	// Main logic
	let unfinishedTasks = [];
	let sourceNote = null;
	let sectionTitle = "Unfinished things from Yesterday...";
	
	// Find the most recent daily note
	const recentNote = findMostRecentNote();
	
	if (recentNote) {
	    try {
	        const noteContent = await app.vault.read(recentNote.file);
	        unfinishedTasks = extractUnfinishedTasks(noteContent);
	        sourceNote = recentNote.fileName;
	        
	        // Generate dynamic section title based on source
	        if (unfinishedTasks.length > 0) {
	            const today = moment(tp.file.title, 'YYYY-MM-DD');
	            const sourceDate = moment(sourceNote.split('-').slice(0, 3).join('-'), 'YYYY-MM-DD');
	            const daysDiff = today.diff(sourceDate, 'days');
	            
	            if (daysDiff === 1) {
	                sectionTitle = "Tasks from yesterday...";
	            } else {
	                const sourceDateFormatted = sourceDate.format('MMMM DD');
	                const sourceDayOfWeek = sourceDate.format('dddd');
	                sectionTitle = `Tasks from ${sourceDateFormatted} (${sourceDayOfWeek})...`;
	            }
	        }
	    } catch (error) {
	        console.log(`Could not read file: ${recentNote.fileName}`);
	    }
	}
	
	// Output the section title
	tR += sectionTitle + '\n';
	
	// Output the unfinished tasks or a placeholder
	if (unfinishedTasks.length > 0) {
	    tR += unfinishedTasks.join('\n');
	} else {
	    tR += '- [ ] ';
	}
	%>
	
	#### 🚀 Things I worked on today...
	- [ ] 
	
	#### ⛅ Things I need to do tomorrow...
	- [ ] 
	
	---
	# 📝 Notes
	
	- <% tp.file.cursor() %>
