# VBA.Callback

## Callback functions in Microsoft Access VBA

Version 1.2.0

*(c) Gustav Brock, Cactus Data ApS, CPH*

### Little known feature of Microsoft Access ###
Callback functions are a hidden gem in Microsoft Access. With these, you can dynamically fill a combobox or listbox entirely from code with a versatility way beyond what a simple static value list can offer. They can be a little hard to get hold on but, once you master the technique, they can meet just about any need no matter how complex.

This series of three articles will cover:

1. The basics of callback functions
2. Making the callback function configurable
3. Sharing one callback function between several controls

This is the first article. The second and third article can be found here:

- Making the callback function configurable
- Sharing one callback function between several controls

---
### 1. Get started ###

For a start, browse the official documentation which is hard to locate if you don't have the magic keywords:

[ComboBox.RowSourceType property (Access)](https://docs.microsoft.com/en-us/office/vba/api/Access.ComboBox.RowSourceType?WT.mc_id=M365-MVP-5002361)

[ListBox.RowSourceType property (Access)](https://docs.microsoft.com/en-us/office/vba/api/Access.ListBox.RowSourceType?WT.mc_id=M365-MVP-5002361)

The information on these pages about this topic is, however, obtuse:

> You can also set the RowSourceType property with a user-defined function. The function name is entered without a preceding equal sign (=) and without the trailing pair of parentheses. You must provide specific function code arguments to tell Access how to fill the control.

However, they both contain a link to a page with useful information:

[RowSourceType property (user-defined function) code argument values](https://docs.microsoft.com/en-us/office/vba/api/access.rowsourcetype?WT.mc_id=M365-MVP-5002361)

This page explains the syntax and provides two examples that can serve as skeletons for your own functions. Study this carefully to get hold of the very special way these functions operate and are called.

Access both calls such a function multiple times and gets values back, thus they are nicknamed **callback** functions, even though *callback* in all other programming languages means something very different.

Each call comes with a value for the parameter **Code** that via the `Select Case` block controls what task the function shall carry out for this call. The main tasks are:

When the form opens:

- Initialise the function
- Set up rows and columns
- Fill rows

When the form closes:

- Option to clean up


#### Example 1 ####

Take the first example from that page, the function that lists the next four Mondays:

```vb
Public Function ListMondays( _
    ByRef ctl As Control, _
    ByVal Id As Long, _
    ByVal Row As Long,
    Byval Column As Long,
    Byval Code As Integer) _
    As Variant

    Dim Offset      As Integer
    Dim WeekdayDate	As Date

    Select Case Code
        Case acLBInitialize     ' Initialize.
            ListMondays = True
        Case acLBOpen           ' Open.
            ListMondays = Timer ' Unique ID.
        Case acLBGetRowCount    ' Get rows.
            ListMondays = 4
        Case acLBGetColumnCount ' Get columns.
            ListMondays = 1
        Case acLBGetColumnWidth ' Get column width.
            ListMondays = -1    ' Use default width.
        Case acLBGetValue       ' Get the data.
            Offset = Abs((9 - Weekday(Date)) Mod 7) 
            WeekdayDate = DateAdd("d", Offset + 7 * row, Date) 
            ListMondays = Format(WeekdayDate, "mmmm d")     
        End Select

End Function
```

For the first calls, it sets up the column and row counts and the column width.
Then, reaching `Case acLBGetValue` it calculates the upcoming Monday and the three subsequent Mondays and formats these for a human friendly display like:


| Monday  |
| :------ |
| June 14 |
| June 21 |
| June 28 |
| July 5  |

While this makes it easy for the user to select the right Monday and, say, **June 21** is what you wish to insert in an e-mail message, it isn't of much use if its **the date** of that Monday you want.

Another limitation of this example is, that the function name is used for returning values. Thus, if you wish to modify the name of the function, no less than seven places must the code be adjusted.

#### Improved example ####

Another example will show how to eliminate these limitations and introduce a few refinements.
This example will list the names of the weekdays of a week and the VBA weekday value of these:

```
' Callback function to list the weekday names of a week.
' The selected value will be a value of enum VbDayOfWeek (Long).
' The displayed values will be those of WeekdayName(DayOfWeek, False, vbSunday).
'
' Example for retrieval of selected value:
'
'   Dim DayOfWeek As VbDayOfWeek
'   DayOfWeek = Me!ControlName.Value
'
' Typical settings for combobox or listbox:
'
'   ControlSource:  Bound or unbound
'   RowSource:      Leave empty
'   RowSourceType:  CallWeekdays
'   BoundColumn:    1
'   LimitToList:    Yes
'   AllowEditing:   No
'   Format:         None
'   ColumnCount:    Don't care. Will be set by the function
'   ColumnWidths:   Don't care. Will be overridden by the function
'
' 2021-02-19. Cactus Data ApS, CPH.
'
Public Function CallWeekdays( _
    ByRef Control As Control, _
    ByVal Id As Long, _
    ByVal Row As Long, _
    ByVal Column As Variant, _
    ByVal Code As Integer) _
    As Variant

    ' Adjustable constants.
    '
    ' Left margin of combobox to align the values on list with the formatted value displayed.
    ' Empiric value.
    Const LeftMargin        As Integer = 23

    ' Fixed constants.
    '
    ' Count of rows to display.
    Const RowCount          As Long = DaysPerWeek
    ' Count of columns in the control.
    Const ColumnCount       As Long = 2

    Static ColumnWidth(0 To ColumnCount - 1) As Integer
    Static FirstDayOfWeek   As VbDayOfWeek

    Dim DayOfWeek           As VbDayOfWeek
    Dim Value               As Variant

    Select Case Code
        Case acLBInitialize
            ' Control settings.
            Control.ColumnCount = ColumnCount   ' Set the colum count of the control.
            ColumnWidth(0) = HiddenColumnWidth  ' Hide the bound (value) column.
            ColumnWidth(1) = DefaultColumnWidth ' Set the width of the display column to the default width.
            If Control.ControlType = acComboBox Then
                Control.LeftMargin = LeftMargin ' Adjust left margin of combobox.
            End If

            ' Value settings.
            FirstDayOfWeek = SystemDayOfWeek    ' First day in the week as to the system settings.

            ' Initialize.
            Value = True                        ' True to initialize.
        Case acLBOpen
            Value = Timer                       ' Autogenerated unique ID.
        Case acLBGetRowCount                    ' Get count of rows.
            Value = RowCount                    ' Set count of rows.
        Case acLBGetColumnCount                 ' Get count of columns.
            Value = ColumnCount                 ' Set count of columns.
        Case acLBGetColumnWidth                 ' Get the column width.
            Value = ColumnWidth(Column)         ' Use preset column widths.
        Case acLBGetValue                       ' Get the data for each row.
            DayOfWeek = (FirstDayOfWeek + Row - 1) Mod DaysPerWeek + 1
            If Column = 0 Then
                ' Return weekday value.
                Value = DayOfWeek
            Else
                ' Return friendly name for display.
                Value = StrConv(WeekdayName(DayOfWeek, False, vbSunday), vbProperCase)
            End If
        Case acLBGetFormat                      ' Format the data.
            ' N/A                               ' Apply the value or the display format.
        Case acLBClose                          ' Do something when the form recalculates or closes.
            ' no-op.
        Case acLBEnd                            ' Do something more when the form recalculates or closes.
            ' no-op.
    End Select

    ' Return Value.
    CallWeekdays = Value

End Function
```

First, it uses constants throughout to avoid "magic numbers" and to ease customisation. Next, it operates with two columns, where the first (the "data column") is the weekday value, and the second is the formatted text to display for the user. `ColumnWidth` is an array holding the column widths of the columns of which the first is zero, thus hidden, as the value from this is for the code of the form only, not for display. The array is static, so its values only need to be specified once.

The setting of the left margin is cosmetic only; it aligns the value displayed in the combobox with the values displayed in the dropdown list for a neater visual appearance of the combobox control during the interaction with the user, when he/she browses the items to make a selection.

You will notice another *static variable*, `FirstDayOfWeek`. These hold their values between the multiple calls of the function and need only to be set once - at the initial call of the function, not for every call of the function, thus speeding up the subsequent calls. For this function, the speed gain isn't that much, but for similar functions, where data are retrieved from stored data, it can be dramatic.

Finally, the first weekday to display is found from the system settings of Windows by this function:

```
' Returns the weekday of the first day of the week according to the current Windows settings.
'
' 2017-05-03. Gustav Brock, Cactus Data ApS, CPH.
'
Public Function SystemDayOfWeek() As VbDayOfWeek

    Const DateOfSaturday    As Date = #12:00:00 AM#

    Dim DayOfWeek   As VbDayOfWeek

    DayOfWeek = vbSunday + vbSaturday - Weekday(DateOfSaturday, vbUseSystemDayOfWeek)

    SystemDayOfWeek = DayOfWeek

End Function
```

Now the combobox can be filled by calculating the weekdays for the first column and, by the function `WeekdayName`, obtaining the friendly (localised) names of these for the second column. There is no format to apply, as the values for the visible column are text.
The dropdown could look like this:

![Example 1](images/WeekdayCombobox.png)

#### Returning date values ####

While returning integers and text from a callback function isn't difficult, it is another matter regarding date and time, as such values also must be applied a format to be displayed.

This, however, is easily solved using a callback function for the combobox or listbox, as it returns a *Variant* which can hold any data type including *Date*. This means that - by using two columns - you can let the first return a true *Date* value and the second (the visible) return a formatted value that fits the purpose of the combobox or listbox on the form.

An example is the following function, which simply lists the days (displayed as Monday, Tuesday, etc.) of the current week and - when a day is selected - returns the date of this as a date value:

```
' Callback function to list the weekday names of a week.
' The selected value will be the date of the weekday in the current week.
' The displayed values will be those of WeekdayName(DayOfWeek, False, vbSunday).
'
' Example for retrieval of selected value:
'
'   Dim DateOfWeek As Date
'   DateOfWeek = Me!ControlName.Value
'
' Typical settings for combobox or listbox:
'
'   ControlSource:  Bound or unbound
'   RowSource:      Leave empty
'   RowSourceType:  CallThisWeekDates
'   BoundColumn:    1
'   LimitToList:    Yes
'   AllowEditing:   No
'   Format:         None
'   ColumnCount:    Don't care. Will be set by the function
'   ColumnWidths:   Don't care. Will be overridden by the function
'
' 2021-02-16. Cactus Data ApS, CPH.
'
Public Function CallThisWeekDates( _
    ByRef Control As Control, _
    ByVal Id As Long, _
    ByVal Row As Long, _
    ByVal Column As Variant, _
    ByVal Code As Integer) _
    As Variant

    ' Adjustable constants.
    '
    ' Left margin of combobox to align the values on list with the formatted value displayed.
    ' Empiric value.
    Const LeftMargin        As Integer = 23
    
    ' Fixed constants.
    '
    ' Count of rows to display.
    Const RowCount          As Long = DaysPerWeek
    ' Count of columns in the control.
    Const ColumnCount       As Integer = 2
    
    Static ColumnWidth(0 To ColumnCount - 1) As Integer
    Static FirstDateOfWeek  As Date
    
    Dim DateOfWeek          As Date
    Dim Value               As Variant
    
    Select Case Code
        Case acLBInitialize
            ' Control settings.
            Control.ColumnCount = ColumnCount   ' Set the colum count of the control.
            ColumnWidth(0) = HiddenColumnWidth  ' Hide the bound (value) column.
            ColumnWidth(1) = DefaultColumnWidth ' Set the width of the display column to the default width.
            If Control.ControlType = acComboBox Then
                Control.LeftMargin = LeftMargin ' Adjust left margin of combobox.
            End If
            
            ' Value settings.
                                                ' First date of the week as to the system settings.
            FirstDateOfWeek = DateThisWeekPrimo(Date, vbUseSystemDayOfWeek)
            
            ' Initialize.
            Value = True                        ' True to initialize.
        Case acLBOpen
            Value = Timer                       ' Autogenerated unique ID.
        Case acLBGetRowCount                    ' Get count of rows.
            Value = RowCount                    ' Set count of rows.
        Case acLBGetColumnCount                 ' Get count of columns.
            Value = ColumnCount                 ' Set count of columns.
        Case acLBGetColumnWidth                 ' Get the column width.
            Value = ColumnWidth(Column)         ' Use preset column widths.
        Case acLBGetValue                       ' Get the data for each row.
            DateOfWeek = DateAdd("d", Row, FirstDateOfWeek)
            If Column = 0 Then
                ' Return date of weekday.
                Value = DateOfWeek
            Else
                ' Return friendly name for display.
                Value = StrConv(Format(DateOfWeek, "dddd"), vbProperCase)
            End If
        Case acLBGetFormat                      ' Format the data.
            ' N/A                               ' Apply the value or the display format.
        Case acLBClose                          ' Do something when the form recalculates or closes.
            ' no-op.
        Case acLBEnd                            ' Do something more when the form recalculates or closes.
            ' no-op.
    End Select
    
    ' Return Value.
    CallThisWeekDates = Value

End Function
```

This function will also return a fixed count of seven rows, but this time the returned value will be the date of the selected weekday.

When initialised, it will find the date for the first weekday of the current week according to the settings of Windows. It uses a supporting function for this, which will not be listed here, but essentially uses this expression:

```
FirstDateOfWeek = DateAdd("d", 1 - Weekday(Date, vbUseSystemDayOfWeek), Date)
```

The value is stored in the static variable `FirstDateOfWeek` and then, for each of the following days, one day is added to that date.

As for the format of the displayed days, this is fixed, and the resulting text - the names of the weekdays - is what will be listed.

#### Select weekday date in a form ####

To turn the function into practical use is quite simple: 

- Create a combobox
- Adjust its settings

The settings to set are few and basic (see in-line comments at top) as the function controls some of the critical settings of the control.

The operation may also be simple. For example, to set the selected date for display in a textbox, use an `AfterUpdate` event like this:

```
Private Sub ThisWeekDates_AfterUpdate()

    Me!WeekdayDate.Value = Me!ThisWeekDates.Value
    
End Sub
```

Here is an example of such a form:

![Example 2](images/ThisWeekDates.png)

This form is included in the demo. Open it, select any weekday, and the date displayed will be of that weekday in the current week no matter which weekday is the first of the week.

---
### 2. Dynamic formatting ###

The previously discussed examples of callback functions have been static in the sense, that - once initialised - they behave the same for as long the form holding the listbox or combobox using it is open.

This is true even if the control is required; contrary to what could be expected, a requery of the control (manually or from code) will not reinitialise the function, only retrieve the generated values again. 
Strangely, to force a true requery of the control and the function it uses, the only method, I've found, is to reassign the `ColumWidths` property of the listbox or combobox:

```
' Requery control.
Control.ColumnWidths = Control.ColumnWidths
```

It doesn't make sense, but it works.

Having this knowledge, the next question is how to pass parameters to the callback function to make it behave differently. The core function of the function will normally not have to be changed - more likely would it be the *count* of items listed, the *format* of these, or the *range* of values, for example the start date and end date for a list of dates.

To pass and read such changes, a special value of the argument `Code` can be used. 

For the operation of the function, different constants are passed to the function, and the `Select Case Code` block then controls which steps to take. Now, study the list of constants:

| Name | Value |
| ---- | ----- |
| acLBInitialize | 0 |
| acLBOpen | 1 |
| acLBGetRowCount | 3 |
| acLBGetColumnCount | 4 |
| acLBGetColumnWidth | 5 |
| acLBGetValue | 6 |
| acLBGetFormat | 7 |
| acLBClose | 8 |
| acLBEnd | 9 |

Apparently, there is no value of 2 in use, thus no constant for this value exists. This is not so, however, only is the value for internal use only - to check for the *RowSource* type of the control.
Running some tests will reveal, that calling a callback function later with a value of 2 for the argument `Code` "does nothing". This opens an option to call the function to do "something else" than it otherwise is supposed to do, and this "something else" could be to reconfigure the callback function.

To set this up, a named constant is introduced for the purpose:

| Name | Value |
| ---- | ----- |
| acLBOpenAsVariant | 2 |

When called with this constant, also an `Id` is passed to the function to distinguish between a default call and later custom call:

```
' Control IDs.
Const DefaultId     As Long = -1
Const ResetId       As Long = 0
Const ActionId      As Long = 1
```

Using `ActionId` is the key to configure the function. What to configure and with which value, is passed via two of the remaining arguments: `Row` and `Column`, for example a *start date*, a *row count*, or a *format*:

```
' Setting IDs.
Const StartDateId   As Long = 1
Const RowCountId    As Long = 2
Const FormatId      As Long = 3
```

The configuration values are passed via `Column` which, for this reason, is declared as `Variant`.

To make all this readable, it is held in three levels of `Select Case` constructs. Here is an example from the function `CallUltimoMonthDates`:

```
Case acLBOpenAsVariant
' Id:       Action.
' Row:      Parameter id.
' Column:   Parameter value.
Select Case Id
    Case DefaultId                              ' Default call.
        If Not Initialized Then
            Start = Date
            Year = VBA.Year(Start)              ' Year of the first month to list.
            Month = VBA.Month(Start) + 1        ' Month of the second month to list.
            RowCount = Years * MonthsPerYear    ' Count of rows to display.
            If Control.ControlType = acComboBox Then
                Format(1) = Control.Format      ' Retrieve the display format from the combobox's Format property.
                Control.LeftMargin = LeftMargin ' Adjust left margin of combobox.
            Else
                Format(1) = ListboxFormat       ' Set the display format for the listbox.
            End If
            Initialized = True
        End If
    Case ResetId                                ' Custom call. Ignore custom settings for the current control.
        Initialized = False
    Case ActionId                               ' Custom call. Set one optional value.
        ' Row:      The id of the parameter to adjust.
        ' Column:   The value of the parameter.
        Select Case Row
            Case StartDateId                    ' Start date.
                Start = Column
                Year = VBA.Year(Start)          ' Year of the first month to list.
                Month = VBA.Month(Start) + 1    ' Month of the second month to list.
            Case RowCountId                     ' Count of weeks to list.
                RowCount = Column
            Case FormatId                       ' Format for display.
                If VarType(Column) = vbString Then
                    Format(1) = Column
                End If
        End Select
End Select
```
It may look a bit convoluted, but the use of meaningful names for the constants makes it easier to follow.

The challenge is to both set the static variables when the function is initialised and to preserve those that should not be altered at a later reconfiguration. If there was no option to preserve them, the function would need to be initialised in full even if only a single parameter was adjusted.
This is controlled by the static variable `Initialized` in the "Default" section of the code. It is set to `True` after the first call, causing this section to be ignored for all later calls.

In the "Reset" section of the code, `Initialized` is reset to False, causing the function to reinitialise at a later call having `Id` set to `DefaultId`.

The third section, "Action", is where the function can be reconfigured using the values from the arguments `Row` and `Column`.
In the function listed here, three parameters can be reconfigured: 

- The start date
- The count of rows to list
- The format of the listed dates

###### Reconfiguration ######

It would be possible to reconfigure the function by a series of calls with different parameters, but it will be difficult to read and maintain. For this reason, a function has been created, that will reconfigure one, two, or all parameters and requery both the control and the function in one go:

```
' Set custom parameters for a ComboBox or a ListBox having the function
' CallUltimoMonthDates as RowsourceType.
'
' Usage, where the parameter Object is a ComboBox or a ListBox object:
'
'   Set start date of list:
'   ConfigUltimoMonthDates Object, , #1/1/2020#
'
'   Set count of dates to list (for ComboBox only, ignoreded for ListBox):
'   ConfigUltimoMonthDates Object, , , 10
'
'   Set all parameters:
'   ConfigUltimoMonthDates Object, #4/1/2000#, 18
'
'   Reset all parameters to default settings.
'   NB: Could (should) be called when unloading the form:
'   ConfigUltimoMonthDates Object
'
' 2021-03-01. Cactus Data ApS, CPH.
'
Public Sub ConfigUltimoMonthDates( _
    ByRef Control As Control, _
    Optional ByVal StartDate As Date, _
    Optional ByVal RowCount As Long, _
    Optional ByVal Format As String)
    
    Const FunctionName  As String = "CallUltimoMonthDates"
    Const NoOpValue     As Long = 0
    Const DefaultId     As Long = -1
    Const ResetId       As Long = 0
    Const ActionId      As Long = 1
    Const StartDateId   As Long = 1
    Const RowCountId    As Long = 2
    Const FormatId      As Long = 3
    
    Dim ControlType     As AcControlType
    Dim SetValue        As Boolean
    
    If Not Control Is Nothing Then
        ControlType = Control.ControlType
        If ControlType = acListBox Or ControlType = acComboBox Then
            If Control.RowSourceType = FunctionName Then
                If RowCount <> NoOpValue Then
                    If Control.ControlType = acListBox Then
                        ' Setting of row count not supported.
                        RowCount = NoOpValue
                    End If
                End If
                
                ' Make sure, that this control has called the callback function to be initialized.
                ' That may not be the case, if this configuration function is called during form loading.
                Application.Run FunctionName, Control, DefaultId, NoOpValue, NoOpValue, acLBOpenAsVariant
                
                ' Set parameter(s) and run the function by its name.
                If DateDiff("d", StartDate, #12:00:00 AM#) <> 0 Then
                    Application.Run FunctionName, Control, ActionId, StartDateId, DateValue(StartDate), acLBOpenAsVariant
                    SetValue = True
                End If
                If RowCount > 0 Then
                    Application.Run FunctionName, Control, ActionId, RowCountId, RowCount, acLBOpenAsVariant
                    SetValue = True
                End If
                If Format <> "" Then
                    Application.Run FunctionName, Control, ActionId, FormatId, Format, acLBOpenAsVariant
                    SetValue = True
                End If
                If Not SetValue = True Then
                    ' Reset to default values.
                    Application.Run FunctionName, Control, ResetId, NoOpValue, NoOpValue, acLBOpenAsVariant
                End If
                    
                ' Apply settings.
                Application.Run FunctionName, Control, DefaultId, NoOpValue, NoOpValue, acLBOpenAsVariant
                ' Requery control.
                Control.ColumnWidths = Control.ColumnWidths
            End If
        End If
    End If
    
End Sub
```

Note, that the three arguments are optional and have been given descriptive names making it easy to reconfigure one or more parameters.
As the last step, the function calls the *magic command* described above that will requery the control and the function.

An example of a combobox using this callback function is shown in the form *CallbackDemoUltimoMonths*:

![Example 3](images/UltimoMonthDates.png)

At top, a groupbox controls if either the ultimo dates of the months of the current year should be listed, or it should be a count of dates from today. If the latter is chosen, the count of rows can be specified.

So, if the groupbox is changed, the start date and the count of rows listed must be set. The code to do this is quite simple - the values are set and, in the last line, the control is passed the revised parameters and requeried by the call to the configuration function:

```
Private Sub UltimoSelect_AfterUpdate()

    Dim StartDate   As Date
    Dim RowCount    As Long
    Dim Enabled     As Boolean
    
    Select Case UltimoSelect.Value
        Case 0
            ' List dates from today.
            StartDate = Date
            ' Allow to adjust the count of months to list.
            RowCount = Me!DateRows.Value
            Enabled = True
        Case 1
            ' List the current year's ultimo month dates.
            StartDate = DateSerial(Year(Date), 1, 1)
            ' Lock row count to the count of months for a year.
            RowCount = MonthsPerYear
            Enabled = False
    End Select
    
    Me!DateRows.Value = RowCount
    Me!DateRows.Enabled = Enabled
    Me!DateRows.Locked = Not Enabled
    
    ConfigUltimoMonthDates Me!UltimoMonthDates, StartDate, RowCount
    
End Sub
```

This is probably as simple as it can get, and an example like the one shown here can easily be adopted for many scenarios.


---
### 3. One function, multiple controls ###

If you use several callback functions in an application, most likely they will be used by individual controls (combobox or listbox). 
Sometimes you may wish to use one callback function for several controls. This will pose no problem as long as the function has a fixed configuration or, if it can be reconfigured, the same configuration should be used for all the controls using this function.
Also, even if configured dynamically, one function can be used for multiple controls, if these are located on individual forms that are not likely to be open at the same time.

However, there can be scenarios where multiple controls are placed on the same form and could use the same callback function but with different configurations.
A simple solution to this could be to duplicate the full code of the function having, say:

- MyCallbackFunction1
- MyCallbackFunction2

Likewise, the configurating functions could be duplicated:

- MyConfigFunction1
- MyConfigFunction2

But it is easy to see the limits of this method.

To do it smarter, the callback function should be able to distinguish between the calling controls and to store the individual configurations needed to server the controls.

An example is the function `CallWeekdayDates` which lists the dates of one weekday in an interval of dates. In addition to listing dates from different date intervals, it is easy to imagine a set up with seven combo- or listboxes each listing dates of one of the seven weekdays of a week.
This is the full function:

```
' Callback function to list the dates of a weekday for a count of weeks.
' By default, the first day of the week according to the system settings
' is listed from the current date for twelve weeks.
'
' Optionally, any weekday, any start date, and any count of weeks can be
' set by the function ConfigWeekdayDates.
' Multiple controls - even a mix of comboboxes and listboxes - can be
' controlled simultaneously with individual settings.
'
' The format of the listed dates are determined by the controls' Format
' properties.
'
' Example for retrieval of selected value:
'
'   Dim SelectedDate As Date
'   SelectedDate = Me!ControlName.Value
'
' Typical settings for combobox or listbox:
'
'   ControlSource:  Bound or unbound
'   RowSource:      Leave empty
'   RowSourceType:  CallWeekdayDates
'   BoundColumn:    1
'   LimitToList:    Yes
'   AllowEditing:   No
'   ColumnCount:    Don't care. Will be set by the function
'   ColumnWidths:   Don't care. Will be overridden by the function
'   ListCount:      Don't care. Will be overridden by the function (ComboBox only)
'   Format:         Optional. A valid format for date values (ComboBox only)
'   Tag:            Optional. 1 to 255.
'                   Count of rows listed. If empty or 0, DefaultWeekCount is used
'
' 2021-03-01. Gustav Brock. Cactus Data ApS, CPH.
'
Public Function CallWeekdayDates( _
    ByRef Control As Control, _
    ByVal Id As Long, _
    ByVal Row As Long, _
    ByVal Column As Variant, _
    ByVal Code As Integer) _
    As Variant
    
    ' Adjustable constants.
    '
    ' Initial count of weeks to list.
    ' Fixed for a listbox. A combobox can be reconfigured with function ConfigWeekdayDates.
    ' Will be overridden by a value specified in property Tag.
    Const DefaultWeekCount      As Integer = 16
    ' Format for the display column in a listbox.
    Const ListboxFormat         As String = "Short Date"
    ' Left margin of combobox to align the values on list with the formatted value displayed.
    ' Empiric value.
    Const LeftMargin            As Integer = 23
    
    ' Fixed constants.
    '
    ' Count of columns in the control.
    Const ColumnCount           As Integer = 2
    
    ' Function constants.
    '
    ' Array constants.
    Const ControlOption         As Integer = 0
    Const ApplyOption           As Integer = 1
    Const WeekdayOption         As Integer = 2
    Const StartDateOption       As Integer = 3
    Const RowCountOption        As Integer = 4
    Const FormatOption          As Integer = 5
    Const ControlDimension      As Integer = 2
    ' Control IDs.
    Const DefaultId             As Long = -1
    Const ResetId               As Long = 0
    Const ActionId              As Long = 1
    ' Setting IDs.
    Const DayOfWeekId           As Long = 1
    Const StartDateId           As Long = 2
    Const RowCountId            As Long = 3
    Const FormatId              As Long = 4
    
    Static ColumnWidth(0 To ColumnCount - 1)    As Integer
    Static DateFormat           As String
    Static OptionalValues()     As Variant
    Static Initialized          As Boolean
    
    Dim ControlName             As String
    Dim ControlIndex            As Integer
    Dim StartDate               As Date
    Dim FirstDate               As Date
    Dim DayOfWeek               As VbDayOfWeek
    Dim RowCount                As Integer
    Dim Value                   As Variant
    
    Select Case Code
        Case acLBInitialize
            ' Control settings.
            Control.ColumnCount = ColumnCount           ' Set the colum count of the control.
            Control.ColumnWidths = MinimalColumnWidth   ' Record width of first column. This is used for a requery.
            ColumnWidth(0) = HiddenColumnWidth          ' Hide the bound (value) column.
            ColumnWidth(1) = DefaultColumnWidth         ' Set the width of the display column to the default width.
            If Control.ControlType = acComboBox Then
                ' Set the date format later.            ' Retrieve the display format from the combobox's Format property.
                Control.LeftMargin = LeftMargin         ' Adjust left margin of combobox.
            Else
                DateFormat = ListboxFormat              ' Set the display format for the listbox.
            End If
            
            If Not Initialized Then
                ' Array for optional values has not been dimmed in this session.
                ReDim OptionalValues(ControlOption To FormatOption, 0 To 0)
                Initialized = True
            End If
            
            ' Initialize.
            Value = True                                ' True to initialize.
        Case acLBOpenAsVariant
            ControlName = Control.Name
            ' Find or create the index of the current control.
            For ControlIndex = LBound(OptionalValues, ControlDimension) To UBound(OptionalValues, ControlDimension)
                If OptionalValues(ControlOption, ControlIndex) = ControlName Then
                    Exit For
                End If
            Next
            If ControlIndex > UBound(OptionalValues, ControlDimension) Then
                ' Add yet a control.
                ReDim Preserve OptionalValues(ControlOption To FormatOption, 0 To ControlIndex)
                OptionalValues(ControlOption, ControlIndex) = ControlName
            End If
            
            ' Id:       Action.
            ' Row:      Parameter id.
            ' Column:   Parameter value.
            Select Case Id
                Case DefaultId                          ' Default call.
                    If OptionalValues(ApplyOption, ControlIndex) = False Then
                        ' Apply initial/default settings.
                        OptionalValues(ControlOption, ControlIndex) = ControlName
                        OptionalValues(ApplyOption, ControlIndex) = True
                        ' Use system's first day of week.
                        OptionalValues(WeekdayOption, ControlIndex) = SystemDayOfWeek
                        OptionalValues(StartDateOption, ControlIndex) = Date
                        RowCount = Val(Control.Tag)
                        If RowCount = 0 Then
                            RowCount = DefaultWeekCount
                        End If
                        OptionalValues(RowCountOption, ControlIndex) = RowCount
                        ' If this is a combobox, retrieve the default format from its Format property.
                        If Control.ControlType = acComboBox Then
                            DateFormat = Control.Format
                        End If
                        OptionalValues(FormatOption, ControlIndex) = DateFormat
                    End If
                Case ResetId                            ' Custom call. Ignore custom settings for the current control.
                    OptionalValues(ApplyOption, ControlIndex) = False
                Case ActionId                           ' Custom call. Set one optional value.
                    ' Row:      The id of the parameter to adjust.
                    ' Column:   The value of the parameter.
                    Select Case Row
                        Case DayOfWeekId                ' Day of week.
                            OptionalValues(WeekdayOption, ControlIndex) = Column
                        Case StartDateId                ' Start date.
                            If VarType(Column) = vbDate Then
                                OptionalValues(StartDateOption, ControlIndex) = Column
                            End If
                        Case RowCountId                 ' Count of weeks to list.
                            OptionalValues(RowCountOption, ControlIndex) = Column
                        Case FormatId                   ' Format for display.
                            If VarType(Column) = vbString Then
                                OptionalValues(FormatOption, ControlIndex) = Column
                            End If
                    End Select
            End Select
            
            ' Do not return a value.
        Case acLBOpen
            ' Value will be rounded to integer, so multiply by 100. Howeever, a Single
            ' (as returned by Timer) will be rounded to even, so convert the value to Long.
            Value = CLng(Timer * 100)                   ' Autogenerated unique ID.
        Case acLBGetRowCount                            ' Get count of rows.
            ControlName = Control.Name
            ' Find the index of the current control.
            For ControlIndex = LBound(OptionalValues, ControlDimension) To UBound(OptionalValues, ControlDimension)
                If OptionalValues(ControlOption, ControlIndex) = ControlName Then
                    Exit For
                End If
            Next
            ' Retrieve current setting.
            RowCount = OptionalValues(RowCountOption, ControlIndex)
            Value = RowCount                            ' Set count of rows.
        Case acLBGetColumnCount                         ' Get count of columns.
            Value = ColumnCount                         ' Set count of columns.
        Case acLBGetColumnWidth                         ' Get the column width.
            Value = ColumnWidth(Column)                 ' Use preset column widths.
        Case acLBGetValue                               ' Get the data for each row.
            ControlName = Control.Name
            ' Find the index of the current control.
            For ControlIndex = LBound(OptionalValues, ControlDimension) To UBound(OptionalValues, ControlDimension)
                If OptionalValues(ControlOption, ControlIndex) = ControlName Then
                    Exit For
                End If
            Next
            ' Retrieve current settings.
            StartDate = OptionalValues(StartDateOption, ControlIndex)
            DayOfWeek = OptionalValues(WeekdayOption, ControlIndex)
            ' Retrieve and save for this ControlIndex the format for the
            ' next call of the function which will have Code = acLBGetFormat.
            DateFormat = OptionalValues(FormatOption, ControlIndex)
            ' Calculate the earliest date later than or equal to the start date.
            FirstDate = DateNextWeekday(DateAdd("d", -1, StartDate), DayOfWeek)
            ' Calculate the date for this row.
            Value = DateAdd("ww", Row, FirstDate)
        Case acLBGetFormat                              ' Format the data.
            If Column = 1 Then
                Value = DateFormat                      ' Apply the value or the display format.
            End If
        Case acLBClose                                  ' Do something when the form recalculates or closes.
            ' no-op.
        Case acLBEnd                                    ' Do something more when the form recalculates or closes.
            ' no-op.
    End Select

    ' Return Value.
    CallWeekdayDates = Value

End Function
```

The crucial part is the array `OptionalValues`. This holds and separates the parameters for each control calling the function.

At first, it is initialised to hold parameters for one control (one dimension: `0 To 0`):

```
If Not Initialized Then
    ' Array for optional values has not been dimmed in this session.
    ReDim OptionalValues(ControlOption To FormatOption, 0 To 0)
    Initialized = True
End If
```            

The count of elements is fixed and will be used to store the option values using these constants for better readability:

```
Const ControlOption         As Integer = 0
Const ApplyOption           As Integer = 1
Const WeekdayOption         As Integer = 2
Const StartDateOption       As Integer = 3
Const RowCountOption        As Integer = 4
Const FormatOption          As Integer = 5
```

Later, when a reconfiguration is called, a check is made if the calling control is known; if not, the array will be redimmed with one dimension more (`ControlIndex`) to also hold this control by its name:

```
ControlName = Control.Name
' Find or create the index of the current control.
For ControlIndex = LBound(OptionalValues, ControlDimension) To UBound(OptionalValues, ControlDimension)
    If OptionalValues(ControlOption, ControlIndex) = ControlName Then
        Exit For
    End If
Next
If ControlIndex > UBound(OptionalValues, ControlDimension) Then
    ' Add yet a control.
    ReDim Preserve OptionalValues(ControlOption To FormatOption, 0 To ControlIndex)
    OptionalValues(ControlOption, ControlIndex) = ControlName
End If
```

From the very first example of a callback function, `Timer` has been used to create a "unique" id for the calling control. `Timer` returns a **Single* with a resolution of about 1/18 second but, for some unknown reason, the internal variable of the function is a *Long*. This will, of course, round the value from `Timer` to the second which isn't enough to separate two or more controls, as the form, when loading, will initialise these asyncronously. To overcome this, the value is multiplied by 100:

```
' Value will be rounded to integer, so multiply by 100. Howeever, a Single
' (as returned by Timer) will be rounded to even, so convert the value to Long.
Value = CLng(Timer * 100)                   ' Autogenerated unique ID.
```

Now, with the `OptionValues` array and a more likely unique `Id`, parameters for several controls can be written and read - even asyncronously - without interfering with each other.

###### Running the function ######

The initial call of the function sets, for each control using the function, the **default* parameters:

```
Select Case Id
    Case DefaultId                          ' Default call.
        If OptionalValues(ApplyOption, ControlIndex) = False Then
            ' Apply initial/default settings.
            OptionalValues(ControlOption, ControlIndex) = ControlName
            OptionalValues(ApplyOption, ControlIndex) = True
            ' Use system's first day of week.
            OptionalValues(WeekdayOption, ControlIndex) = SystemDayOfWeek
            OptionalValues(StartDateOption, ControlIndex) = Date
            RowCount = Val(Control.Tag)
            If RowCount = 0 Then
                RowCount = DefaultWeekCount
            End If
            OptionalValues(RowCountOption, ControlIndex) = RowCount
            ' If this is a combobox, retrieve the default format from its Format property.
            If Control.ControlType = acComboBox Then
                DateFormat = Control.Format
            End If
            OptionalValues(FormatOption, ControlIndex) = DateFormat
        End If
```

If later to be reconfigured, the function's *action* section is called. The dimension is controlled by `ControlIndex` which has been set to that of the control, and then the element holding the parameter can be set:

```
    Case ActionId                           ' Custom call. Set one optional value.
        ' Row:      The id of the parameter to adjust.
        ' Column:   The value of the parameter.
        Select Case Row
            Case DayOfWeekId                ' Day of week.
                OptionalValues(WeekdayOption, ControlIndex) = Column
            Case StartDateId                ' Start date.
                OptionalValues(StartDateOption, ControlIndex) = Column
            Case RowCountId                 ' Count of weeks to list.
                OptionalValues(RowCountOption, ControlIndex) = Column
            Case FormatId                   ' Format for display.
                If VarType(Column) = vbString Then
                    OptionalValues(FormatOption, ControlIndex) = Column
                End If
        End Select
```
If more than one parameter should be set, the function is called multiple time with different values for `Row` to match the elements of the array to be set.

If the calls to reconfigure the function are made from its *configuration function* (see later), there should be no reason to validate the values. If needed, an example of basic validation is shown for the *format* option.

Having the parameters set - default or customised - these are read when the values to be listed are retrieved and formatted:

```
Case acLBGetValue                               ' Get the data for each row.
    ControlName = Control.Name
    ' Find the index of the current control.
    For ControlIndex = LBound(OptionalValues, ControlDimension) To UBound(OptionalValues, ControlDimension)
        If OptionalValues(ControlOption, ControlIndex) = ControlName Then
            Exit For
        End If
    Next
    ' Retrieve current settings.
    StartDate = OptionalValues(StartDateOption, ControlIndex)
    DayOfWeek = OptionalValues(WeekdayOption, ControlIndex)
    ' Retrieve and save for this ControlIndex the format for the
    ' next call of the function which will have Code = acLBGetFormat.
    DateFormat = OptionalValues(FormatOption, ControlIndex)
    ' Calculate the earliest date later than or equal to the start date.
    FirstDate = DateNextWeekday(DateAdd("d", -1, StartDate), DayOfWeek)
    ' Calculate the date for this row.
    Value = DateAdd("ww", Row, FirstDate)
Case acLBGetFormat                              ' Format the data.
    If Column = 1 Then
        Value = DateFormat                      ' Apply the value or the display format.
    End If
```

When creating the date, a supporting function, `DateNextWeekday`, is used to find the first weekday following the specified start date. Essentially, it performs this calculation:

```
FirstDate = DateAdd("d", 7 - (Weekday(FromDate, DayOfWeek) - 1), FromDate)
```

where `FromDate` is the day before `StartDate` to have `StartDate` included on the list of dates.

The format is only handled for column 1 as the first column (column 0) holds the true date value which has no format.

###### Reconfiguration ######

As for the previous callback function, a function, `ConfigWeekdayDates`, has been created that will reconfigure one, two, or all parameters and requery a control and the function in one go. It is very similar to the previous function, so it will not be listed or discussed here.

###### Multiple controls in action ######

To demonstrate the setup of multiple controls on one form, the form *CallbackDemoWeekdayDates* has been created. It contains one listbox and two comboboxes:

![Example 4](images/WeekdayDates.png)

The listbox is set to list dates of the current weekday, and the two comboboxes are set to list dates of the first and the last weekday respectively.

A tiny detail is, that the *Tag* property of the comboboxes can be set to hold the default count of dates (rows) to be listed. This can be set later, as can the date range to be listed, and this will interact with the row count.
The format can be adjusted as well by modifying the content of the *Format* textbox.

At load of the form, the three controls are reconfigured. It takes a little, but is not convoluted, so please study the in-line comments and notice the usage of the configuration function:

```
Private Sub Form_Load()

    Const DefaultRowCount   As Integer = 16

    Dim FirstDate           As Date
    Dim RowCount            As Long
    Dim DayOfWeek           As VbDayOfWeek
    
    ' Assign the first day of the week as the default selection.
    Me!Weekdays1.Value = SystemDayOfWeek
    Me!StartDate1.Value = Date
    ' Assign the last day of the week as the default selection.
    Me!Weekdays2.Value = (SystemDayOfWeek - 1 - 1 + DaysPerWeek) Mod DaysPerWeek + 1
    Me!StartDate2.Value = Date
    
    ' Adjust count of rows to be listed if set in property Tag.
    RowCount = Val(Me!WeekdayDates1.Tag)
    If RowCount = 0 Then
        RowCount = DefaultRowCount
    End If
    Me!DateRows1.Value = RowCount
    
    RowCount = Val(Me!WeekdayDates2.Tag)
    If RowCount = 0 Then
        RowCount = DefaultRowCount
    End If
    Me!DateRows2.Value = RowCount
    
    ' Calculate end dates to display.
    FirstDate = DateNextWeekday(DateAdd("d", -1, Me!StartDate1.Value), Me!Weekdays1.Value)
    Me!EndDate1.Value = DateAdd("ww", RowCount - 1, FirstDate)

    FirstDate = DateNextWeekday(DateAdd("d", -1, Me!StartDate2.Value), Me!Weekdays2.Value)
    Me!EndDate2.Value = DateAdd("ww", RowCount - 1, FirstDate)
    
    ' Retrieve and display the default date format from the comboboxes' Format property.
    Me!FormatDate1.DefaultValue = """" & Me!WeekdayDates1.Format & """"
    Me!FormatDate2.DefaultValue = """" & Me!WeekdayDates2.Format & """"
    
    ' Check if default weekday of the weekday selector is different from SystemDayOfWeek.
    DayOfWeek = Me!Weekdays1.Value
    If DayOfWeek <> SystemDayOfWeek Then
        ' Reconfig the list/combobox as the weekday to list is different from SystemDayOfWeek.
        ConfigWeekdayDates Me!WeekdayDates1, DayOfWeek
    End If
    DayOfWeek = Me!Weekdays2.Value
    If DayOfWeek <> SystemDayOfWeek Then
        ' Reconfig the list/combobox as the weekday to list is different from SystemDayOfWeek.
        ConfigWeekdayDates Me!WeekdayDates2, DayOfWeek
    End If

    DayOfWeek = Weekday(Date)
    If DayOfWeek <> SystemDayOfWeek Then
        ' Reconfig the list/combobox as the weekday to list is different from SystemDayOfWeek.
        ConfigWeekdayDates Me!ListWeekdayDates, DayOfWeek
    End If

End Sub
```

Though not necessary if the controls are reconfigured when loading the form, it may be a good idea to reset the parameters for the control when closing the form. This is very easy to do, passing the control as the only argument to the configuration function:

```
Private Sub Form_Unload(Cancel As Integer)

    ConfigWeekdayDates Me!ListWeekdayDates
    ConfigWeekdayDates Me!WeekdayDates1
    ConfigWeekdayDates Me!WeekdayDates2
    
End Sub
```

The form has a lot of code to control the interaction between the controls supporting the three listbox and comboboxes. This is, however, very specific for this form. For most other forms, the setup and context will be very different, so the details will not be listed or discussed here. Study the code to get inspiration - all steps are carefully commented.

### Documentation

Top level documentation generated by [MZ-Tools](https://www.mztools.com/) is included for [Microsoft Access](https://htmlpreview.github.io?https://github.com/GustavBrock/VBA.Calback/blob/master/documentation/CallbackDemo.htm).

Detailed documentation is in-line. 

Full documentation can be found here:

![EE Logo](images/EE%20Logo.png)

[Callback functions in Microsoft Access VBA. Part 1](https://www.experts-exchange.com/articles/35774)

[Callback functions in Microsoft Access VBA. Part 2](https://www.experts-exchange.com/articles/35775)

[Callback functions in Microsoft Access VBA. Part 3](https://www.experts-exchange.com/articles/35776)

---

*If you wish to support my work or need extended support or advice, feel free to:*

<p>

[<img src="https://raw.githubusercontent.com/GustavBrock/VBA.Callback/master/images/BuyMeACoffee.png">](https://www.buymeacoffee.com/gustav/)