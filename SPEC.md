# Daily Todo - Specification

## Project Overview
- **Type**: Single-page web application (HTML/CSS/JS)
- **Core Functionality**: A minimalist daily task manager with add, delete, and complete features
- **Data Storage**: localStorage

## UI/UX Specification

### Design Direction: "Zen Paper" - Japanese-inspired minimalism
- Elegant, calm aesthetic reminiscent of traditional Japanese stationery
- Subtle paper texture and warm, natural tones

### Color Palette
```css
--bg-primary: #F7F5F0        /* Warm ivory paper */
--bg-secondary: #EFECE6      /* Slightly darker paper */
--text-primary: #2C2C2C      /* Rich black ink */
--text-secondary: #7A7A7A    /* Faded gray text */
--accent: #B85C38            /* Vermillion seal red */
--accent-light: #E8D5C4      /* Soft terracotta */
--success: #6B8F71           /* Sage green */
--border: #D4CFC7            /* Subtle warm gray */
--shadow: rgba(44, 44, 44, 0.08)
```

### Typography
- **Display/Title**: "Noto Serif SC" - elegant Chinese serif
- **Body**: "Noto Sans SC" - clean, readable sans-serif  
- Fallback: Georgia, serif

### Layout
- Centered card (max-width: 520px)
- Generous padding (48px horizontal, 40px vertical)
- Soft shadows and rounded corners (12px)

### Components

#### Header
- App title "每日待办" in display font
- Current date displayed elegantly below
- Subtle decorative line separator

#### Input Area
- Clean input field with bottom border only (no box)
- Placeholder text in secondary color
- Add button: minimal "+" icon, vermillion accent on hover

#### Task List
- Individual task items with subtle bottom border
- Custom checkbox: circle that fills with green when checked
- Completed tasks: strikethrough text, muted color
- Delete button (×) appears on hover with smooth fade-in

#### Empty State
- Gentle message when no tasks: "今日无事一身轻"
- Subtle illustration or decorative element

### Animations
- Task appear: slide down + fade in (0.3s ease-out)
- Task complete: checkbox fill animation, strikethrough slides in
- Task delete: slide left + fade out (0.25s)
- Hover states: subtle scale and shadow transitions

### Responsive
- Mobile (< 600px): Full width with 16px margin
- Desktop: Centered card with shadow

## Functionality Specification

### Core Features
1. **Add Task**: Enter text, press Enter or click add button
2. **Delete Task**: Click × button to remove task permanently
3. **Complete Task**: Click checkbox to toggle completion status
4. **Persist Data**: All tasks saved to localStorage automatically

### User Interactions
- Input: Type task → Enter key or click add button
- Complete: Click anywhere on task row except delete button
- Delete: Hover over task → click × icon

### Data Structure
```javascript
{
  id: number,
  text: string,
  completed: boolean,
  createdAt: timestamp
}
```

### Edge Cases
- Empty input: Prevent adding, show subtle shake animation
- Whitespace-only: Trim and ignore
- Duplicate prevention: Not required (allow same tasks)

## Acceptance Criteria

### Visual Checkpoints
- [ ] Warm paper-like background visible
- [ ] Title uses elegant serif font
- [ ] Input field has minimal bottom-border style
- [ ] Checkboxes are custom circular design
- [ ] Completed tasks show strikethrough with muted text
- [ ] Delete button appears smoothly on hover

### Functional Checkpoints
- [ ] Can add new task via Enter key
- [ ] Can add new task via click button
- [ ] Can mark task as complete/incomplete
- [ ] Can delete a task
- [ ] Data persists after page refresh
- [ ] Empty input is handled gracefully

### Animation Checkpoints
- [ ] New tasks animate in smoothly
- [ ] Completed state has visual transition
- [ ] Delete animation works correctly