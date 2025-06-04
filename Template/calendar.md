```dataviewjs
// University Hub Calendar Widget
const currentDate = new Date();
let displayMonth = currentDate.getMonth();
let displayYear = currentDate.getFullYear();
const DB_FILE_NAME = 'calendar-events.md';

async function getOrCreateDbFile() {
    let dbFile = app.vault.getAbstractFileByPath(DB_FILE_NAME);
    if (!dbFile) {
        const initialContent = `# Calendar Events Database\n\nThis file stores all calendar events. Do not edit manually - use the calendar interface.\n\n## Events Data\n\n\`\`\`json\n[]\n\`\`\`\n\n## Events Summary\nTotal events: 0\nLast updated: ${new Date().toLocaleString()}\n`;
        dbFile = await app.vault.create(DB_FILE_NAME, initialContent);
    }
    return dbFile;
}

async function readEventsFromDb() {
    try {
        const dbFile = await getOrCreateDbFile();
        const content = await app.vault.read(dbFile);
        const jsonMatch = content.match(/```json\n([\s\S]*?)\n```/);
        if (jsonMatch) {
            const jsonStr = jsonMatch[1].trim();
            if (jsonStr) return JSON.parse(jsonStr);
        }
        return [];
    } catch (error) {
        console.error('Error reading events database:', error);
        return [];
    }
}

async function writeEventsToDb(eventsList) {
    try {
        const dbFile = await getOrCreateDbFile();
        const content = `# Calendar Events Database\n\nThis file stores all calendar events. Do not edit manually - use the calendar interface.\n\n## Events Data\n\n\`\`\`json\n${JSON.stringify(eventsList, null, 2)}\n\`\`\`\n\n## Events Summary\nTotal events: ${eventsList.length}\nLast updated: ${new Date().toLocaleString()}\n`;
        await app.vault.modify(dbFile, content);
    } catch (error) {
        console.error('Error writing events database:', error);
        throw error;
    }
}

async function getEventsForDate(date) {
    const dateStr = date.toISOString().split('T')[0];
    const allEvents = await readEventsFromDb();
    return allEvents.filter(event => event.date === dateStr).map(event => ({
        ...event,
        color: getEventColor(event.type)
    }));
}

function getEventColor(type) {
    const colors = { 'due': '#dc2626', 'deadline': '#ea580c', 'exam': '#7c2d12', 'class': '#059669', 'event': '#059669', 'reminder': '#6366f1', 'note': '#4f46e5' };
    return colors[type] || '#059669';
}

function showAddEventWidget(selectedDate) {
    const dateStr = selectedDate.toISOString().split('T')[0];
    const formattedDate = selectedDate.toLocaleDateString('en-US', { weekday: 'long', year: 'numeric', month: 'long', day: 'numeric' });
    
    const overlay = document.createElement('div');
    overlay.style.cssText = `position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: rgba(0, 0, 0, 0.5); z-index: 1000; display: flex; justify-content: center; align-items: center;`;
    
    const modal = document.createElement('div');
    modal.style.cssText = `background: var(--background-primary); border-radius: 12px; padding: 24px; width: 450px; max-width: 90vw; box-shadow: 0 20px 25px -5px rgba(0, 0, 0, 0.1); border: 1px solid var(--background-modifier-border);`;
    
    modal.innerHTML = `
        <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; padding-bottom: 16px; border-bottom: 1px solid var(--background-modifier-border);">
            <h3 style="margin: 0; color: var(--text-normal); font-size: 1.2em; font-weight: 600;">Add Event - ${formattedDate}</h3>
            <button id="closeBtn" style="background: none; border: none; font-size: 1.5em; cursor: pointer; color: var(--text-muted); padding: 4px 8px; border-radius: 4px;">×</button>
        </div>
        
        <form id="eventForm" style="display: flex; flex-direction: column; gap: 16px;">
            <div style="display: flex; flex-direction: column; gap: 6px;">
                <label style="font-weight: 500; color: var(--text-normal); font-size: 0.9em;">Event Title *</label>
                <input id="titleInput" type="text" required placeholder="Enter event title..." style="padding: 8px 12px; border: 1px solid var(--background-modifier-border); border-radius: 6px; background: var(--background-primary); color: var(--text-normal); font-size: 0.9em;">
            </div>
            
            <div style="display: flex; flex-direction: column; gap: 6px;">
                <label style="font-weight: 500; color: var(--text-normal); font-size: 0.9em;">Event Type *</label>
                <select id="typeSelect" required style="padding: 10px 12px; border: 1px solid var(--background-modifier-border); border-radius: 6px; background: var(--background-primary); color: var(--text-normal); font-size: 0.9em; line-height: 1.4; height: auto; min-height: 40px;">
                    <option value="due">Assignment Due</option>
                    <option value="deadline">Project Deadline</option>
                    <option value="exam">Exam</option>
                    <option value="class">Class/Lecture</option>
                    <option value="event">General Event</option>
                    <option value="reminder">Reminder</option>
                </select>
            </div>
            
            <div style="display: flex; flex-direction: column; gap: 6px;">
                <label style="font-weight: 500; color: var(--text-normal); font-size: 0.9em;">Time (optional)</label>
                <input id="timeInput" type="time" style="padding: 8px 12px; border: 1px solid var(--background-modifier-border); border-radius: 6px; background: var(--background-primary); color: var(--text-normal); font-size: 0.9em;">
            </div>
            
            <div style="display: flex; flex-direction: column; gap: 6px;">
                <label style="font-weight: 500; color: var(--text-normal); font-size: 0.9em;">Description (optional)</label>
                <textarea id="descInput" rows="3" placeholder="Add notes or description..." style="padding: 8px 12px; border: 1px solid var(--background-modifier-border); border-radius: 6px; background: var(--background-primary); color: var(--text-normal); font-size: 0.9em; resize: vertical; font-family: inherit;"></textarea>
            </div>
            
            <div style="display: flex; gap: 12px; justify-content: flex-end; margin-top: 20px; padding-top: 16px; border-top: 1px solid var(--background-modifier-border);">
                <button id="cancelBtn" type="button" style="padding: 8px 16px; border: 1px solid var(--background-modifier-border); border-radius: 6px; background: var(--background-primary); color: var(--text-muted); cursor: pointer; font-size: 0.9em;">Cancel</button>
                <button type="submit" style="padding: 8px 16px; border: none; border-radius: 6px; background: var(--interactive-accent); color: white; cursor: pointer; font-weight: 500; font-size: 0.9em;">Save Event</button>
            </div>
        </form>
    `;
    
    overlay.appendChild(modal);
    document.body.appendChild(overlay);
    
    const closeBtn = modal.querySelector('#closeBtn');
    const cancelBtn = modal.querySelector('#cancelBtn');
    const form = modal.querySelector('#eventForm');
    const titleInput = modal.querySelector('#titleInput');
    
    function closeModal() { document.body.removeChild(overlay); }
    
    closeBtn.addEventListener('click', closeModal);
    cancelBtn.addEventListener('click', closeModal);
    overlay.addEventListener('click', (e) => { if (e.target === overlay) closeModal(); });
    
    form.addEventListener('submit', async (e) => {
        e.preventDefault();
        const eventTitle = modal.querySelector('#titleInput').value.trim();
        const eventType = modal.querySelector('#typeSelect').value;
        const eventTime = modal.querySelector('#timeInput').value;
        const eventDesc = modal.querySelector('#descInput').value.trim();
        
        if (!eventTitle) return;
        
        try {
            const allEventsList = await readEventsFromDb();
            const newEvent = {
                id: Date.now().toString(),
                title: eventTitle,
                date: dateStr,
                type: eventType,
                time: eventTime || null,
                description: eventDesc || null,
                created: new Date().toISOString()
            };
            allEventsList.push(newEvent);
            await writeEventsToDb(allEventsList);
            closeModal();
            updateCalendar();
            new Notice(`Event "${eventTitle}" added successfully!`);
        } catch (error) {
            console.error('Error creating event:', error);
            new Notice('Error creating event. Please try again.');
        }
    });
    
    setTimeout(() => titleInput.focus(), 100);
}

async function showEditEventWidget(eventToEdit) {
    const formattedDate = new Date(eventToEdit.date + 'T00:00:00').toLocaleDateString('en-US', { weekday: 'long', year: 'numeric', month: 'long', day: 'numeric' });
    
    const overlay = document.createElement('div');
    overlay.style.cssText = `position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: rgba(0, 0, 0, 0.5); z-index: 1000; display: flex; justify-content: center; align-items: center;`;
    
    const modal = document.createElement('div');
    modal.style.cssText = `background: var(--background-primary); border-radius: 12px; padding: 24px; width: 450px; max-width: 90vw; box-shadow: 0 20px 25px -5px rgba(0, 0, 0, 0.1); border: 1px solid var(--background-modifier-border);`;
    
    modal.innerHTML = `
        <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; padding-bottom: 16px; border-bottom: 1px solid var(--background-modifier-border);">
            <h3 style="margin: 0; color: var(--text-normal); font-size: 1.2em; font-weight: 600;">Edit Event - ${formattedDate}</h3>
            <button id="closeBtn" style="background: none; border: none; font-size: 1.5em; cursor: pointer; color: var(--text-muted); padding: 4px 8px; border-radius: 4px;">×</button>
        </div>
        
        <form id="eventForm" style="display: flex; flex-direction: column; gap: 16px;">
            <div style="display: flex; flex-direction: column; gap: 6px;">
                <label style="font-weight: 500; color: var(--text-normal); font-size: 0.9em;">Event Title *</label>
                <input id="titleInput" type="text" required value="${eventToEdit.title || ''}" style="padding: 8px 12px; border: 1px solid var(--background-modifier-border); border-radius: 6px; background: var(--background-primary); color: var(--text-normal); font-size: 0.9em;">
            </div>
            
            <div style="display: flex; flex-direction: column; gap: 6px;">
                <label style="font-weight: 500; color: var(--text-normal); font-size: 0.9em;">Event Type *</label>
                <select id="typeSelect" required style="padding: 10px 12px; border: 1px solid var(--background-modifier-border); border-radius: 6px; background: var(--background-primary); color: var(--text-normal); font-size: 0.9em; line-height: 1.4; height: auto; min-height: 40px;">
                    <option value="due" ${eventToEdit.type === 'due' ? 'selected' : ''}>Assignment Due</option>
                    <option value="deadline" ${eventToEdit.type === 'deadline' ? 'selected' : ''}>Project Deadline</option>
                    <option value="exam" ${eventToEdit.type === 'exam' ? 'selected' : ''}>Exam</option>
                    <option value="class" ${eventToEdit.type === 'class' ? 'selected' : ''}>Class/Lecture</option>
                    <option value="event" ${eventToEdit.type === 'event' ? 'selected' : ''}>General Event</option>
                    <option value="reminder" ${eventToEdit.type === 'reminder' ? 'selected' : ''}>Reminder</option>
                </select>
            </div>
            
            <div style="display: flex; flex-direction: column; gap: 6px;">
                <label style="font-weight: 500; color: var(--text-normal); font-size: 0.9em;">Time (optional)</label>
                <input id="timeInput" type="time" value="${eventToEdit.time || ''}" style="padding: 8px 12px; border: 1px solid var(--background-modifier-border); border-radius: 6px; background: var(--background-primary); color: var(--text-normal); font-size: 0.9em;">
            </div>
            
            <div style="display: flex; flex-direction: column; gap: 6px;">
                <label style="font-weight: 500; color: var(--text-normal); font-size: 0.9em;">Description (optional)</label>
                <textarea id="descInput" rows="3" style="padding: 8px 12px; border: 1px solid var(--background-modifier-border); border-radius: 6px; background: var(--background-primary); color: var(--text-normal); font-size: 0.9em; resize: vertical; font-family: inherit;">${eventToEdit.description || ''}</textarea>
            </div>
            
            <div style="display: flex; gap: 12px; justify-content: space-between; margin-top: 20px; padding-top: 16px; border-top: 1px solid var(--background-modifier-border);">
                <button id="deleteBtn" type="button" style="padding: 8px 16px; border: 1px solid #dc2626; border-radius: 6px; background: transparent; color: #dc2626; cursor: pointer; font-size: 0.9em;">Delete Event</button>
                <div style="display: flex; gap: 12px;">
                    <button id="cancelBtn" type="button" style="padding: 8px 16px; border: 1px solid var(--background-modifier-border); border-radius: 6px; background: var(--background-primary); color: var(--text-muted); cursor: pointer; font-size: 0.9em;">Cancel</button>
                    <button type="submit" style="padding: 8px 16px; border: none; border-radius: 6px; background: var(--interactive-accent); color: white; cursor: pointer; font-weight: 500; font-size: 0.9em;">Update Event</button>
                </div>
            </div>
        </form>
    `;
    
    overlay.appendChild(modal);
    document.body.appendChild(overlay);
    
    const closeBtn = modal.querySelector('#closeBtn');
    const cancelBtn = modal.querySelector('#cancelBtn');
    const deleteBtn = modal.querySelector('#deleteBtn');
    const form = modal.querySelector('#eventForm');
    const titleInput = modal.querySelector('#titleInput');
    
    function closeModal() { document.body.removeChild(overlay); }
    
    closeBtn.addEventListener('click', closeModal);
    cancelBtn.addEventListener('click', closeModal);
    overlay.addEventListener('click', (e) => { if (e.target === overlay) closeModal(); });
    
    deleteBtn.addEventListener('click', async () => {
        if (confirm('Are you sure you want to delete this event?')) {
            try {
                const allEventsList = await readEventsFromDb();
                const updatedEventsList = allEventsList.filter(e => e.id !== eventToEdit.id);
                await writeEventsToDb(updatedEventsList);
                closeModal();
                updateCalendar();
                new Notice('Event deleted successfully!');
            } catch (error) {
                console.error('Error deleting event:', error);
                new Notice('Error deleting event. Please try again.');
            }
        }
    });
    
    form.addEventListener('submit', async (e) => {
        e.preventDefault();
        const eventTitle = modal.querySelector('#titleInput').value.trim();
        const eventType = modal.querySelector('#typeSelect').value;
        const eventTime = modal.querySelector('#timeInput').value;
        const eventDesc = modal.querySelector('#descInput').value.trim();
        
        if (!eventTitle) return;
        
        try {
            const allEventsList = await readEventsFromDb();
            const eventIndex = allEventsList.findIndex(e => e.id === eventToEdit.id);
            if (eventIndex !== -1) {
                allEventsList[eventIndex] = {
                    ...allEventsList[eventIndex],
                    title: eventTitle,
                    type: eventType,
                    time: eventTime || null,
                    description: eventDesc || null,
                    modified: new Date().toISOString()
                };
                await writeEventsToDb(allEventsList);
                closeModal();
                updateCalendar();
                new Notice('Event updated successfully!');
            } else {
                throw new Error('Event not found');
            }
        } catch (error) {
            console.error('Error updating event:', error);
            new Notice('Error updating event. Please try again.');
        }
    });
    
    setTimeout(() => titleInput.focus(), 100);
}

const calendarContainer = dv.el('div', '', {
    attr: {
        style: `
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
            max-width: 800px;
            margin: 0 auto;
            background: transparent;
            padding: 0 8px;
        `
    }
});

function updateCalendar() {
    const monthNames = ['January', 'February', 'March', 'April', 'May', 'June', 'July', 'August', 'September', 'October', 'November', 'December'];
    const dayNames = ['Sunday', 'Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday'];
    
    calendarContainer.innerHTML = '';
    
    const header = dv.el('div', '', {
        attr: { 
            style: `
                display: flex; 
                align-items: center;
                margin: 0.6em 0 1em;
                padding: 0 8px;
                width: 100%;
            ` 
        }
    });
    
    const prevBtn = dv.el('div', '', {
        attr: { 
            style: `
                align-items: center;
                cursor: pointer;
                display: flex;
                justify-content: center;
                width: 24px;
                color: var(--text-muted);
                transition: color 0.1s ease-in;
            ` 
        }
    });
    
    prevBtn.innerHTML = `<svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><polyline points="15,18 9,12 15,6"></polyline></svg>`;
    
    prevBtn.addEventListener('mouseenter', () => {
        prevBtn.style.color = 'var(--text-normal)';
    });
    prevBtn.addEventListener('mouseleave', () => {
        prevBtn.style.color = 'var(--text-muted)';
    });
    
    const monthTitle = dv.el('h2', '', {
        attr: { 
            style: `
                color: var(--text-normal);
                font-size: 1.5em;
                margin: 0;
                text-align: center;
                flex: 1;
            ` 
        }
    });
    
    const monthSpan = dv.el('span', monthNames[displayMonth], {
        attr: {
            style: `
                font-weight: 500;
                text-transform: capitalize;
            `
        }
    });
    
    const yearSpan = dv.el('span', ` ${displayYear}`, {
        attr: {
            style: `
                color: var(--interactive-accent);
            `
        }
    });
    
    monthTitle.appendChild(monthSpan);
    monthTitle.appendChild(yearSpan);
    
    const nextBtn = dv.el('div', '', {
        attr: { 
            style: `
                align-items: center;
                cursor: pointer;
                display: flex;
                justify-content: center;
                width: 24px;
                color: var(--text-muted);
                transition: color 0.1s ease-in;
                transform: rotate(180deg);
            ` 
        }
    });
    
    nextBtn.innerHTML = `<svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><polyline points="15,18 9,12 15,6"></polyline></svg>`;
    
    nextBtn.addEventListener('mouseenter', () => {
        nextBtn.style.color = 'var(--text-normal)';
    });
    nextBtn.addEventListener('mouseleave', () => {
        nextBtn.style.color = 'var(--text-muted)';
    });
    
    header.appendChild(prevBtn);
    header.appendChild(monthTitle);
    header.appendChild(nextBtn);
    calendarContainer.appendChild(header);
    
    const calendarTable = dv.el('table', '', {
        attr: { 
            style: `
                border-collapse: collapse;
                width: 100%;
                border: none;
            ` 
        }
    });
    
    // Create header row
    const headerRow = dv.el('tr', '');
    
    dayNames.forEach(day => {
        const dayHeader = dv.el('th', day.substring(0, 3).toUpperCase(), {
            attr: { 
                style: `
                    background-color: transparent;
                    color: var(--text-muted);
                    font-size: 0.6em;
                    letter-spacing: 1px;
                    padding: 4px;
                    text-transform: uppercase;
                    text-align: center;
                    font-weight: 600;
                    border: none;
                ` 
            }
        });
        headerRow.appendChild(dayHeader);
    });
    
    calendarTable.appendChild(headerRow);
    
    // Calculate calendar layout
    const firstDay = new Date(displayYear, displayMonth, 1);
    const lastDay = new Date(displayYear, displayMonth + 1, 0);
    const firstDayWeekday = firstDay.getDay();
    const daysInMonth = lastDay.getDate();
    
    // Create calendar rows
    let currentDay = 1;
    let currentRow = dv.el('tr', '');
    
    // Add empty cells for days before the first day of the month
    for (let i = 0; i < firstDayWeekday; i++) {
        const emptyCell = dv.el('td', '', {
            attr: { 
                style: `
                    height: 80px;
                    vertical-align: top;
                    padding: 0;
                    border: none;
                ` 
            }
        });
        currentRow.appendChild(emptyCell);
    }
    
    // Add days of the month
    for (let day = 1; day <= daysInMonth; day++) {
        const date = new Date(displayYear, displayMonth, day);
        const isToday = date.toDateString() === currentDate.toDateString();
        
        const dayCell = dv.el('td', '', {
            attr: {
                style: `
                    height: 80px;
                    vertical-align: top;
                    padding: 0;
                    position: relative;
                    border: none;
                `
            }
        });
        
        const dayContent = dv.el('div', '', {
            attr: {
                style: `
                    background-color: transparent;
                    border-radius: 4px;
                    color: ${isToday ? 'var(--interactive-accent)' : 'var(--text-normal)'};
                    cursor: pointer;
                    font-size: 0.8em;
                    height: 100%;
                    padding: 4px;
                    position: relative;
                    text-align: center;
                    transition: background-color 0.1s ease-in, color 0.1s ease-in;
                    vertical-align: baseline;
                `
            }
        });
        
        dayContent.addEventListener('mouseenter', () => {
            dayContent.style.backgroundColor = 'var(--interactive-hover)';
        });
        dayContent.addEventListener('mouseleave', () => {
            dayContent.style.backgroundColor = 'transparent';
        });
        
        const dayNumber = dv.el('div', day.toString(), {
            attr: { 
                style: `
                    font-weight: 500;
                    margin-bottom: 4px;
                ` 
            }
        });
        
        dayContent.appendChild(dayNumber);
        
        // Add events asynchronously
        (async () => {
            try {
                const dayEventsList = await getEventsForDate(date);
                
                dayEventsList.slice(0, 3).forEach(eventItem => {
                    const eventWidget = dv.el('div', '', {
                        attr: {
                            style: `
                                background: ${eventItem.color}15;
                                border-radius: 4px;
                                padding: 2px 4px;
                                margin: 1px 0;
                                font-size: 0.65em;
                                font-weight: 500;
                                color: var(--text-normal);
                                cursor: pointer;
                                transition: background-color 0.1s ease-in;
                                overflow: hidden;
                                text-overflow: ellipsis;
                                white-space: nowrap;
                                position: relative;
                            `
                        }
                    });
                    
                    // Add a small colored indicator
                    const indicator = dv.el('div', '', {
                        attr: {
                            style: `
                                width: 4px;
                                height: 4px;
                                background: ${eventItem.color};
                                border-radius: 50%;
                                display: inline-block;
                                margin-right: 4px;
                                vertical-align: middle;
                            `
                        }
                    });
                    
                    const eventText = dv.el('span', eventItem.title, {
                        attr: {
                            style: `
                                vertical-align: middle;
                            `
                        }
                    });
                    
                    eventWidget.appendChild(indicator);
                    eventWidget.appendChild(eventText);
                    
                    eventWidget.addEventListener('mouseenter', () => {
                        eventWidget.style.backgroundColor = `${eventItem.color}25`;
                    });
                    
                    eventWidget.addEventListener('mouseleave', () => {
                        eventWidget.style.backgroundColor = `${eventItem.color}15`;
                    });
                    
                    eventWidget.addEventListener('click', (e) => {
                        e.stopPropagation();
                        showEditEventWidget(eventItem);
                    });
                    
                    dayContent.appendChild(eventWidget);
                });
                
                if (dayEventsList.length > 3) {
                    const moreWidget = dv.el('div', `+${dayEventsList.length - 3}`, {
                        attr: { 
                            style: `
                                background: var(--background-modifier-border);
                                border-radius: 4px;
                                padding: 2px 4px;
                                margin: 1px 0;
                                font-size: 0.6em;
                                color: var(--text-muted);
                                text-align: center;
                                cursor: pointer;
                                transition: background-color 0.1s ease-in;
                            ` 
                        }
                    });
                    
                    moreWidget.addEventListener('mouseenter', () => {
                        moreWidget.style.backgroundColor = 'var(--interactive-hover)';
                    });
                    moreWidget.addEventListener('mouseleave', () => {
                        moreWidget.style.backgroundColor = 'var(--background-modifier-border)';
                    });
                    
                    dayContent.appendChild(moreWidget);
                }
            } catch (error) {
                console.error('Error loading events for date:', error);
            }
        })();
        
        // Click handler to show add event widget
        dayContent.addEventListener('click', () => {
            showAddEventWidget(date);
        });
        
        dayCell.appendChild(dayContent);
        currentRow.appendChild(dayCell);
        
        // Start a new row if we've reached Sunday (day 0) and it's not the last day
        if ((firstDayWeekday + day) % 7 === 0 && day < daysInMonth) {
            calendarTable.appendChild(currentRow);
            currentRow = dv.el('tr', '');
        }
    }
    
    // Fill remaining cells in the last row
    while (currentRow.children.length < 7) {
        const emptyCell = dv.el('td', '', {
            attr: { 
                style: `
                    height: 80px;
                    vertical-align: top;
                    padding: 0;
                    border: none;
                ` 
            }
        });
        currentRow.appendChild(emptyCell);
    }
    
    calendarTable.appendChild(currentRow);
    
    // Add alternating row backgrounds for better week differentiation
    const allRows = calendarTable.querySelectorAll('tr');
    allRows.forEach((row, index) => {
        if (index > 0) { // Skip header row
            const isEvenWeek = (index - 1) % 2 === 0;
            row.style.backgroundColor = isEvenWeek ? 'transparent' : 'var(--background-secondary)';
        }
    });
    calendarContainer.appendChild(calendarTable);
    
    // Navigation event listeners
    prevBtn.addEventListener('click', () => {
        displayMonth--;
        if (displayMonth < 0) {
            displayMonth = 11;
            displayYear--;
        }
        updateCalendar();
    });
    
    nextBtn.addEventListener('click', () => {
        displayMonth++;
        if (displayMonth > 11) {
            displayMonth = 0;
            displayYear++;
        }
        updateCalendar();
    });
}

updateCalendar();
calendarContainer;