--[[
QR Code Encoder -------------  © Callum Brieske ------------- NSL NZ

This notice must not be removed. For licensing contact callum@xicontrols.co.nz.

SAMPLE LUA SCRIPT CODE: The following code is strictly provided as an
example of some of the capabilities of Q-SYS.  It is NOT provided as a finished
product, suitable for deployment in specific applications.  Use of any part of
this code is at the user’s own risk. Please read the disclaimer at the bottom of
this source code, it limits the author’s responsibility and liability related to this code.
====================================================================================
--]]

--[[
  Todo:
  - Some efficiency improvements may be possible in the masking operations.
        Perhaps we can dump the table rows to strings & use gmatch?
  - Arrange functions in order of execution.
  - Maybe enforce compliance with ISO 8859-1. Chars should be 0x20 - 0x7E or 0x0A - 0xFF.
  - Align the parameters of each function for cleaner chaining.

  Processing steps:
  1.  Get version & ecLevel for the supplied string.                        qr:calculateVersion( data (string) , ecLevel (int), callback (func) )                           callback passes: data (string), version (int), ecLevelValid (int)
  2.  Add header bytes to the string, and return code groups in a table.    qr:formatData( data (string), version (int), ecLevel (int), callback (func) )                   callback passes: groups (table)
  3.  Split data into the required groups.                                  qr:ecGroups( data (table), callback (func) )                                                    callback passes: groups (table)
          For each block, calculate error correction blocks:                qr:calculateEC( block (table), numberOfEcCodewords, callback )                                  callback passes: block (table)
  4.  Calculate the penalty of each matrix, and return the lowest.          qr:getMatrix( groups (table), version (int), ecLevel (int), callback (func) )                   callback passes: matrix (table)
          Pack the data into a matrix:                                      qr:buildMatrix( groups (table), version (int), ecLevel (int), callback (func) )                 callback passes: matrix (table)
          Apply the mask & calculate the penalty:                           qr:addMask( matrix (table), mask (int) )                                                        callback passes: matrix (table), penalty (int)

  5. We are done building the QR Code! We can now call one (or more) of the following:
          Generate a string that can be printed in the debug window:        qr:generateGridString( matrix (table), border (bool), callback (func) )                         callback passes: string (string)
          Scale the matrix by 'factor':                                     qr:scale( matrix (table), factor (int), callback (func) )                                       callback passes: matrix (table)
          or
          Calculate a scaling factor and call 'qr:scale':                   qr:setRes( matrix (table), horizontalResolution (int), callback (func) )                        callback passes: matrix (table)
          Generate a bitmap image from a matrix. (Scale matrix first)       qr:generateBitmap( matrix (table) )                                                             callback passes: bitmap (binary)
--]]

PluginVersion = "0.0.5"
Infobox = ("QR Code Generator v%s\r\n© Callum Brieske"):format(PluginVersion)
PluginInfo =
{
    Name = ("Callum Brieske~QR Code Generator v%s"):format(PluginVersion),
    Version = PluginVersion,
    Id = "cdc4ffe9-b0eb-4940-af18-bfd2bbceb997",
    Description = "QR Code Generator",
    ShowDebug = false,
}

color = {
	white = { 255, 255, 255 },
	black = { 0, 0, 0 },
	transparent = { 255, 255, 255, 0 }
}

function GetColor()
    return { 0, 0, 0 }
end

function GetPrettyName()
    return ("QR Code Generator\r\nv%s\r\n© Callum Brieske"):format(PluginVersion)
end

function GetProperties()
    return {
		{
			Name = "QR Code Count",
			Type = "integer",
			Min  = 1,
			Max = 64,
			Value = 1
		},
		{
			Name = "CPU Usage",
			Type = "enum",
			Choices = { "Low", "Medium", "High" },
			Value = "High"
		}
	}
end

function GetPages( props )
	local pages = {}
	for i = 1, props["QR Code Count"].Value do
		table.insert( pages, { name = props["QR Code Count"].Value > 1 and ("QR Code %d"):format(i) or "QR Code" } )
	end
    return pages
end

function GetControls( props )
	local controls = {}

	for _, name in ipairs{ "uci", "nic", "wifi_SSID", "wifi_key", "wifi_encryption", "str", "ec_level", "version" } do
		table.insert( controls, {
			Name = name,
			ControlType = "Text",
			Count = props["QR Code Count"].Value,
			UserPin = true,
			PinStyle = "Both"
		})
	end
	for _, name in ipairs{ "type_html_uci", "type_iOS_uci", "type_wifi", "type_string", "wifi_hidden" } do
		table.insert(controls, {
			Name = name,
			ControlType = "Button",
			ButtonType = "Toggle",
			Count = props["QR Code Count"].Value,
			UserPin = true,
			PinStyle = "Both"
		})
	end
	table.insert(controls, {
		Name = "status",
		ControlType = "Indicator",
		IndicatorType = "Status",
		Count = props["QR Code Count"].Value,
		UserPin = true,
		PinStyle = "Output"
	})
	table.insert(controls, {
		Name = "image",
		ControlType = "Button",
		ButtonType = "Trigger",
		Count = props["QR Code Count"].Value,
		UserPin = true,
		PinStyle = "Output"
	})

    return controls
end

function GetControlLayout( props )
	local layout, graphics = {}, {}

	local x, y = 0, 0
	do
		table.insert( graphics, {
			Type = "GroupBox",
			Text = props["QR Code Count"].Value > 1 and ("QR Code %d"):format( props["page_index"].Value ) or "QR Code",
			HTextAlign = "Center",
			StrokeWidth = 1,
			CornerRadius = 8,
			FontSize = 20,
			Font="Roboto",
			FontStyle="Bold",
			Position = { x + 0, y + 0 },
			Size = { 853, 400 }
		})

		-- Options --
		table.insert( graphics, {
			Type = "GroupBox",
			Text = "Options",
			HTextAlign = "Left",
			StrokeWidth = 1,
			CornerRadius = 8,
			FontSize = 14,
			Font="Roboto",
			FontStyle="Regular",
			Position = { x + 10, y + 40 },
			Size = { 240, 350 }
		})
		table.insert( graphics, {
			Type = "Label",
			Text = "QR Code Type",
			FontSize = 14,
			Font="Roboto",
			FontStyle="Regular",
			Position = { x + 20, y + 60 },
			Size = { 220, 18 }
		})
		table.insert( graphics, {
			Type = "Label",
			Text = "HTML5 UCI",
			HTextAlign = "Left",
			FontSize = 14,
			Font="Roboto",
			FontStyle="Regular",
			Position = { x + 50, y + 83 },
			Size = { 190, 18 }
		})
		layout[props["QR Code Count"].Value > 1 and ("type_html_uci %d"):format( props["page_index"].Value ) or "type_html_uci"] = {
			PrettyName = props["QR Code Count"].Value > 1 and ("QR Code %d~Type: HTML5 UCI"):format( props["page_index"].Value ) or "Type: HTML5 UCI",
			Position = { x + 20, y + 83 },
			Size = { 20, 20 },
			Color = color.white,
			UnlinkOffColor = true,
			OffColor = color.white,
			StrokeColor = color.black,
		}
		table.insert( graphics, {
			Type = "Label",
			Text = "iOS UCI",
			HTextAlign = "Left",
			FontSize = 14,
			Font="Roboto",
			FontStyle="Regular",
			Position = { x + 50, y + 108 },
			Size = { 190, 18 }
		})
		layout[props["QR Code Count"].Value > 1 and ("type_iOS_uci %d"):format( props["page_index"].Value ) or "type_iOS_uci"] = {
			PrettyName = props["QR Code Count"].Value > 1 and ("QR Code %d~Type: iOS UCI"):format( props["page_index"].Value ) or "Type: iOS UCI",
			Position = { x + 20, y + 108 },
			Size = { 20, 20 },
			Color = color.white,
			UnlinkOffColor = true,
			OffColor = color.white,
			StrokeColor = color.black
		}
		table.insert( graphics, {
			Type = "Label",
			Text = "Wifi Network",
			HTextAlign = "Left",
			FontSize = 14,
			Font="Roboto",
			FontStyle="Regular",
			Position = { x + 50, y + 133 },
			Size = { 190, 18 }
		})
		layout[props["QR Code Count"].Value > 1 and ("type_wifi %d"):format( props["page_index"].Value ) or "type_wifi"] = {
			PrettyName = props["QR Code Count"].Value > 1 and ("QR Code %d~Type: Wifi"):format( props["page_index"].Value ) or "Type: Wifi",
			Position = { x + 20, y + 133 },
			Size = { 20, 20 },
			Color = color.white,
			UnlinkOffColor = true,
			OffColor = color.white,
			StrokeColor = color.black
		}
		table.insert( graphics, {
			Type = "Label",
			Text = "Custom String",
			HTextAlign = "Left",
			FontSize = 14,
			Font="Roboto",
			FontStyle="Regular",
			Position = { x + 50, y + 158 },
			Size = { 190, 18 }
		})
		layout[props["QR Code Count"].Value > 1 and ("type_string %d"):format( props["page_index"].Value ) or "type_string"] = {
			PrettyName = props["QR Code Count"].Value > 1 and ("QR Code %d~Type: Custom String"):format( props["page_index"].Value ) or "Type: Custom String",
			Position = { x + 20, y + 158 },
			Size = { 20, 20 },
			Color = color.white,
			UnlinkOffColor = true,
			OffColor = color.white,
			StrokeColor = color.black
		}
		table.insert( graphics, {
			Type = "Label",
			Text = "Error Correction Level",
			FontSize = 14,
			Font="Roboto",
			FontStyle="Regular",
			Position = { x + 20, y + 186 },
			Size = { 220, 18 }
		})
		layout[props["QR Code Count"].Value > 1 and ("ec_level %d"):format( props["page_index"].Value ) or "ec_level"] = {
			PrettyName = props["QR Code Count"].Value > 1 and ("QR Code %d~Error Correction Level"):format( props["page_index"].Value ) or "Error Correction Level",
			Position = { x + 20, y + 204 },
			Size = { 220, 18 },
			Style = "ComboBox"
		}
		table.insert( graphics, {
			Type = "Label",
			Text = "QR Code Size",
			FontSize = 14,
			Font="Roboto",
			FontStyle="Regular",
			Position = { x + 20, y + 230 },
			Size = { 220, 18 }
		})
		layout[props["QR Code Count"].Value > 1 and ("version %d"):format( props["page_index"].Value ) or "version"] = {
			PrettyName = props["QR Code Count"].Value > 1 and ("QR Code %d~QR Code Size"):format( props["page_index"].Value ) or "QR Code Size",
			Position = { x + 20, y + 248 },
			Size = { 220, 18 },
			Style = "ComboBox"
		}
		table.insert( graphics, {
			Type = "Label",
			Text = "Status",
			FontSize = 14,
			Font="Roboto",
			FontStyle="Regular",
			Position = { x + 20, y + 274 },
			Size = { 220, 18 }
		})
		layout[props["QR Code Count"].Value > 1 and ("status %d"):format( props["page_index"].Value ) or "status"] = {
			PrettyName = props["QR Code Count"].Value > 1 and ("QR Code %d~Status"):format( props["page_index"].Value ) or "Status",
			Style = "Text",
			Position = { x + 20, y + 292 },
			Size = { 220, 18 }
		}
		table.insert( graphics, {
			Type = "Label",
			Text = Infobox,
			VTextAlign = "Bottom",
			FontSize = 14,
			Font="Roboto",
			FontStyle="Regular",
			Position = { x + 20, y + 318 },
			Size = { 220, 62 }
		})

		-- UCI --
		table.insert( graphics, {
			Type = "GroupBox",
			Text = "UCI",
			HTextAlign = "Left",
			StrokeWidth = 1,
			CornerRadius = 8,
			FontSize = 14,
			Font="Roboto",
			FontStyle="Regular",
			Position = { x + 260, y + 40 },
			Size = { 240, 110 }
		})
		table.insert( graphics, {
			Type = "Label",
			Text = "UCI",
			FontSize = 14,
			Font="Roboto",
			FontStyle="Regular",
			Position = { x + 270, y + 60 },
			Size = { 220, 18 }
		})
		layout[props["QR Code Count"].Value > 1 and ("uci %d"):format( props["page_index"].Value ) or "uci"] = {
			PrettyName = props["QR Code Count"].Value > 1 and ("QR Code %d~UCI"):format( props["page_index"].Value ) or "UCI",
			Position = { x + 270, y + 78 },
			Size = { 220, 18 },
			Style = "ComboBox"
		}
		table.insert( graphics, {
			Type = "Label",
			Text = "Network Interface",
			FontSize = 14,
			Font="Roboto",
			FontStyle="Regular",
			Position = { x + 270, y + 104 },
			Size = { 220, 18 }
		})
		layout[props["QR Code Count"].Value > 1 and ("nic %d"):format( props["page_index"].Value ) or "nic"] = {
			PrettyName = props["QR Code Count"].Value > 1 and ("QR Code %d~NIC"):format( props["page_index"].Value ) or "NIC",
			Position = { x + 270, y + 122 },
			Size = { 220, 18 },
			Style = "ComboBox"
		}

		-- Wifi --
		table.insert( graphics, {
			Type = "GroupBox",
			Text = "Wifi",
			HTextAlign = "Left",
			StrokeWidth = 1,
			CornerRadius = 8,
			FontSize = 14,
			Font="Roboto",
			FontStyle="Regular",
			Position = { x + 260, y + 160 },
			Size = { 240, 154 }
		})
		table.insert( graphics, {
			Type = "Label",
			Text = "SSID",
			FontSize = 14,
			Font="Roboto",
			FontStyle="Regular",
			Position = { x + 270, y + 180 },
			Size = { 220, 18 }
		})
		layout[props["QR Code Count"].Value > 1 and ("wifi_SSID %d"):format( props["page_index"].Value ) or "wifi_SSID"] = {
			PrettyName = props["QR Code Count"].Value > 1 and ("QR Code %d~Wifi SSID"):format( props["page_index"].Value ) or "Wifi SSID",
			Position = { x + 270, y + 198 },
			Size = { 220, 18 }
		}
		table.insert( graphics, {
			Type = "Label",
			Text = "Passcode",
			FontSize = 14,
			Font="Roboto",
			FontStyle="Regular",
			Position = { x + 270, y + 224 },
			Size = { 220, 18 }
		})
		layout[props["QR Code Count"].Value > 1 and ("wifi_key %d"):format( props["page_index"].Value ) or "wifi_key"] = {
			PrettyName = props["QR Code Count"].Value > 1 and ("QR Code %d~Wifi Passcode"):format( props["page_index"].Value ) or "Wifi Passcode",
			Position = { x + 270, y + 242 },
			Size = { 220, 18 }
		}
		table.insert( graphics, {
			Type = "Label",
			Text = "Encryption",
			FontSize = 14,
			Font="Roboto",
			FontStyle="Regular",
			Position = { x + 270, y + 268 },
			Size = { 110, 18 }
		})
		layout[props["QR Code Count"].Value > 1 and ("wifi_encryption %d"):format( props["page_index"].Value ) or "wifi_encryption"] = {
			PrettyName = props["QR Code Count"].Value > 1 and ("QR Code %d~Wifi Encryption"):format( props["page_index"].Value ) or "Wifi Encryption",
			Position = { x + 270, y + 286 },
			Size = { 110, 18 },
			Style = "ComboBox"
		}
		table.insert( graphics, {
			Type = "Label",
			Text = "Hidden",
			FontSize = 14,
			Font="Roboto",
			FontStyle="Regular",
			Position = { x + 380, y + 268 },
			Size = { 110, 18 }
		})
		layout[props["QR Code Count"].Value > 1 and ("wifi_hidden %d"):format( props["page_index"].Value ) or "wifi_hidden"] = {
			PrettyName = props["QR Code Count"].Value > 1 and ("QR Code %d~Wifi Hidden"):format( props["page_index"].Value ) or "Wifi Hidden",
			Position = { x + 395, y + 286 },
			Size = { 81, 18 }
		}

		-- Custom String --
		table.insert( graphics, {
			Type = "GroupBox",
			Text = "Custom",
			HTextAlign = "Left",
			StrokeWidth = 1,
			CornerRadius = 8,
			FontSize = 14,
			Font="Roboto",
			FontStyle="Regular",
			Position = { x + 260, y + 324 },
			Size = { 240, 66 }
		})
		table.insert( graphics, {
			Type = "Label",
			Text = "String",
			FontSize = 14,
			Font="Roboto",
			FontStyle="Regular",
			Position = { x + 270, y + 344 },
			Size = { 220, 18 }
		})
		layout[props["QR Code Count"].Value > 1 and ("str %d"):format( props["page_index"].Value ) or "str"] = {
			PrettyName = props["QR Code Count"].Value > 1 and ("QR Code %d~String"):format( props["page_index"].Value ) or "String",
			Position = { x + 270, y + 362 },
			Size = { 220, 18 }
		}

		-- QR Code --
		table.insert( graphics, {
			Type = "GroupBox",
			Text = "QR Code",
			HTextAlign = "Left",
			StrokeWidth = 1,
			CornerRadius = 8,
			FontSize = 14,
			Font="Roboto",
			FontStyle="Regular",
			Position = { x + 510, y + 40 },
			Size = { 333, 350 }
		})
		layout[props["QR Code Count"].Value > 1 and ("image %d"):format( props["page_index"].Value ) or "image"] = {
			PrettyName = props["QR Code Count"].Value > 1 and ("QR Code %d~QR Code Button"):format( props["page_index"].Value ) or "QR Code Button",
			Position = { x + 520, y + 67 },
			Size = { 313, 313 },
			ButtonVisualStyle = "Flat",
			Margin = 0,
			Padding = 0,
			CornerRadius = 0,
			StrokeWidth = 0,
			Color = color.white,
			UnlinkOffColor = true,
			OffColor = color.white
		}
	end

    return layout, graphics
end

if Controls then

	local qr = {

		-- Specifications for QR codes as per ISO/IEC 18004:2015
		qrSpec = {
			{ capacity = {   19,   16,   13,    9 }, ver =      0, groups = { {  1,    }, {  1,    }, {  1,    }, {  1,    } }, codewords = { { {  19,   7 }               }, { {  16,  10 }               }, { {  13,  13 }               }, { {   9,  17 }               } }, align = {                                    }, },
			{ capacity = {   34,   28,   22,   16 }, ver =      0, groups = { {  1,    }, {  1,    }, {  1,    }, {  1,    } }, codewords = { { {  34,  10 }               }, { {  28,  16 }               }, { {  22,  22 }               }, { {  16,  28 }               } }, align = {   6,  18                           }, },
			{ capacity = {   55,   44,   34,   26 }, ver =      0, groups = { {  1,    }, {  1,    }, {  2,    }, {  2,    } }, codewords = { { {  55,  15 }               }, { {  44,  26 }               }, { {  17,  18 }               }, { {  13,  22 }               } }, align = {   6,  22                           }, },
			{ capacity = {   80,   64,   48,   36 }, ver =      0, groups = { {  1,    }, {  2,    }, {  2,    }, {  4,    } }, codewords = { { {  80,  20 }               }, { {  32,  18 }               }, { {  24,  26 }               }, { {   9,  16 }               } }, align = {   6,  26                           }, },
			{ capacity = {  108,   86,   62,   46 }, ver =      0, groups = { {  1,    }, {  2,    }, {  2,  2 }, {  2,  2 } }, codewords = { { { 108,  26 }               }, { {  43,  24 }               }, { {  15,  18 }, {  16,  18 } }, { {  11,  22 }, {  12,  22 } } }, align = {   6,  30                           }, },
			{ capacity = {  136,  108,   76,   60 }, ver =      0, groups = { {  2,    }, {  4,    }, {  4,    }, {  4,    } }, codewords = { { {  68,  18 }               }, { {  27,  16 }               }, { {  19,  24 }               }, { {  15,  28 }               } }, align = {   6,  34                           }, },
			{ capacity = {  156,  124,   88,   66 }, ver =  31892, groups = { {  2,    }, {  4,    }, {  2,  4 }, {  4,  1 } }, codewords = { { {  78,  20 }               }, { {  31,  18 }               }, { {  14,  18 }, {  15,  18 } }, { {  13,  26 }, {  14,  26 } } }, align = {   6,  22,  38                      }, },
			{ capacity = {  194,  154,  110,   86 }, ver =  34236, groups = { {  2,    }, {  2,  2 }, {  4,  2 }, {  4,  2 } }, codewords = { { {  97,  24 }               }, { {  38,  22 }, {  39,  22 } }, { {  18,  22 }, {  19,  22 } }, { {  14,  26 }, {  15,  26 } } }, align = {   6,  24,  42                      }, },
			{ capacity = {  232,  182,  132,  100 }, ver =  39577, groups = { {  2,    }, {  3,  2 }, {  4,  4 }, {  4,  4 } }, codewords = { { { 116,  30 }               }, { {  36,  22 }, {  37,  22 } }, { {  16,  20 }, {  17,  20 } }, { {  12,  24 }, {  13,  24 } } }, align = {   6,  26,  46                      }, },
			{ capacity = {  274,  216,  154,  122 }, ver =  42195, groups = { {  2,  2 }, {  4,  1 }, {  6,  2 }, {  6,  2 } }, codewords = { { {  68,  18 }, {  69,  18 } }, { {  43,  26 }, {  44,  26 } }, { {  19,  24 }, {  20,  24 } }, { {  15,  28 }, {  16,  28 } } }, align = {   6,  28,  50                      }, },
			{ capacity = {  324,  254,  180,  140 }, ver =  48118, groups = { {  4,    }, {  1,  4 }, {  4,  4 }, {  3,  8 } }, codewords = { { {  81,  20 }               }, { {  50,  30 }, {  51,  30 } }, { {  22,  28 }, {  23,  28 } }, { {  12,  24 }, {  13,  24 } } }, align = {   6,  30,  54                      }, },
			{ capacity = {  370,  290,  206,  158 }, ver =  51042, groups = { {  2,  2 }, {  6,  2 }, {  4,  6 }, {  7,  4 } }, codewords = { { {  92,  24 }, {  93,  24 } }, { {  36,  22 }, {  37,  22 } }, { {  20,  26 }, {  21,  26 } }, { {  14,  28 }, {  15,  28 } } }, align = {   6,  32,  58                      }, },
			{ capacity = {  428,  334,  244,  180 }, ver =  55367, groups = { {  4,    }, {  8,  1 }, {  8,  4 }, { 12,  4 } }, codewords = { { { 107,  26 }               }, { {  37,  22 }, {  38,  22 } }, { {  20,  24 }, {  21,  24 } }, { {  11,  22 }, {  12,  22 } } }, align = {   6,  34,  62                      }, },
			{ capacity = {  461,  365,  261,  197 }, ver =  58893, groups = { {  3,  1 }, {  4,  5 }, { 11,  5 }, { 11,  5 } }, codewords = { { { 115,  30 }, { 116,  30 } }, { {  40,  24 }, {  41,  24 } }, { {  16,  20 }, {  17,  20 } }, { {  12,  24 }, {  13,  24 } } }, align = {   6,  26,  46,  66                 }, },
			{ capacity = {  523,  415,  295,  223 }, ver =  63784, groups = { {  5,  1 }, {  5,  5 }, {  5,  7 }, { 11,  7 } }, codewords = { { {  87,  22 }, {  88,  22 } }, { {  41,  24 }, {  42,  24 } }, { {  24,  30 }, {  25,  30 } }, { {  12,  24 }, {  13,  24 } } }, align = {   6,  26,  48,  70                 }, },
			{ capacity = {  589,  453,  325,  253 }, ver =  68472, groups = { {  5,  1 }, {  7,  3 }, { 15,  2 }, {  3, 13 } }, codewords = { { {  98,  24 }, {  99,  24 } }, { {  45,  28 }, {  46,  28 } }, { {  19,  24 }, {  20,  24 } }, { {  15,  30 }, {  16,  30 } } }, align = {   6,  26,  50,  74                 }, },
			{ capacity = {  647,  507,  367,  283 }, ver =  70749, groups = { {  1,  5 }, { 10,  1 }, {  1, 15 }, {  2, 17 } }, codewords = { { { 107,  28 }, { 108,  28 } }, { {  46,  28 }, {  47,  28 } }, { {  22,  28 }, {  23,  28 } }, { {  14,  28 }, {  15,  28 } } }, align = {   6,  30,  54,  78                 }, },
			{ capacity = {  721,  563,  397,  313 }, ver =  76311, groups = { {  5,  1 }, {  9,  4 }, { 17,  1 }, {  2, 19 } }, codewords = { { { 120,  30 }, { 121,  30 } }, { {  43,  26 }, {  44,  26 } }, { {  22,  28 }, {  23,  28 } }, { {  14,  28 }, {  15,  28 } } }, align = {   6,  30,  56,  82                 }, },
			{ capacity = {  795,  627,  445,  341 }, ver =  79154, groups = { {  3,  4 }, {  3, 11 }, { 17,  4 }, {  9, 16 } }, codewords = { { { 113,  28 }, { 114,  28 } }, { {  44,  26 }, {  45,  26 } }, { {  21,  26 }, {  22,  26 } }, { {  13,  26 }, {  14,  26 } } }, align = {   6,  30,  58,  86                 }, },
			{ capacity = {  861,  669,  485,  385 }, ver =  84390, groups = { {  3,  5 }, {  3, 13 }, { 15,  5 }, { 15, 10 } }, codewords = { { { 107,  28 }, { 108,  28 } }, { {  41,  26 }, {  42,  26 } }, { {  24,  30 }, {  25,  30 } }, { {  15,  28 }, {  16,  28 } } }, align = {   6,  34,  62,  90                 }, },
			{ capacity = {  932,  714,  512,  406 }, ver =  87683, groups = { {  4,  4 }, { 17,    }, { 17,  6 }, { 19,  6 } }, codewords = { { { 116,  28 }, { 117,  28 } }, { {  42,  26 }               }, { {  22,  28 }, {  23,  28 } }, { {  16,  30 }, {  17,  30 } } }, align = {   6,  28,  50,  72,  94            }, },
			{ capacity = { 1006,  782,  568,  442 }, ver =  92361, groups = { {  2,  7 }, { 17,    }, {  7, 16 }, { 34,    } }, codewords = { { { 111,  28 }, { 112,  28 } }, { {  46,  28 }               }, { {  24,  30 }, {  25,  30 } }, { {  13,  24 }               } }, align = {   6,  26,  50,  74,  98            }, },
			{ capacity = { 1094,  860,  614,  464 }, ver =  96236, groups = { {  4,  5 }, {  4, 14 }, { 11, 14 }, { 16, 14 } }, codewords = { { { 121,  30 }, { 122,  30 } }, { {  47,  28 }, {  48,  28 } }, { {  24,  30 }, {  25,  30 } }, { {  15,  30 }, {  16,  30 } } }, align = {   6,  30,  54,  78, 102            }, },
			{ capacity = { 1174,  914,  664,  514 }, ver = 102084, groups = { {  6,  4 }, {  6, 14 }, { 11, 16 }, { 30,  2 } }, codewords = { { { 117,  30 }, { 118,  30 } }, { {  45,  28 }, {  46,  28 } }, { {  24,  30 }, {  25,  30 } }, { {  16,  30 }, {  17,  30 } } }, align = {   6,  28,  54,  80, 106            }, },
			{ capacity = { 1276, 1000,  718,  538 }, ver = 102881, groups = { {  8,  4 }, {  8, 13 }, {  7, 22 }, { 22, 13 } }, codewords = { { { 106,  26 }, { 107,  26 } }, { {  47,  28 }, {  48,  28 } }, { {  24,  30 }, {  25,  30 } }, { {  15,  30 }, {  16,  30 } } }, align = {   6,  32,  58,  84, 110            }, },
			{ capacity = { 1370, 1062,  754,  596 }, ver = 110507, groups = { { 10,  2 }, { 19,  4 }, { 28,  6 }, { 33,  4 } }, codewords = { { { 114,  28 }, { 115,  28 } }, { {  46,  28 }, {  47,  28 } }, { {  22,  28 }, {  23,  28 } }, { {  16,  30 }, {  17,  30 } } }, align = {   6,  30,  58,  86, 114            }, },
			{ capacity = { 1468, 1128,  808,  628 }, ver = 110734, groups = { {  8,  4 }, { 22,  3 }, {  8, 26 }, { 12, 28 } }, codewords = { { { 122,  30 }, { 123,  30 } }, { {  45,  28 }, {  46,  28 } }, { {  23,  30 }, {  24,  30 } }, { {  15,  30 }, {  16,  30 } } }, align = {   6,  34,  62,  90, 118            }, },
			{ capacity = { 1531, 1193,  871,  661 }, ver = 117786, groups = { {  3, 10 }, {  3, 23 }, {  4, 31 }, { 11, 31 } }, codewords = { { { 117,  30 }, { 118,  30 } }, { {  45,  28 }, {  46,  28 } }, { {  24,  30 }, {  25,  30 } }, { {  15,  30 }, {  16,  30 } } }, align = {   6,  26,  50,  74,  98, 122       }, },
			{ capacity = { 1631, 1267,  911,  701 }, ver = 119615, groups = { {  7,  7 }, { 21,  7 }, {  1, 37 }, { 19, 26 } }, codewords = { { { 116,  30 }, { 117,  30 } }, { {  45,  28 }, {  46,  28 } }, { {  23,  30 }, {  24,  30 } }, { {  15,  30 }, {  16,  30 } } }, align = {   6,  30,  54,  78, 102, 126       }, },
			{ capacity = { 1735, 1373,  985,  745 }, ver = 126325, groups = { {  5, 10 }, { 19, 10 }, { 15, 25 }, { 23, 25 } }, codewords = { { { 115,  30 }, { 116,  30 } }, { {  47,  28 }, {  48,  28 } }, { {  24,  30 }, {  25,  30 } }, { {  15,  30 }, {  16,  30 } } }, align = {   6,  26,  52,  78, 104, 130       }, },
			{ capacity = { 1843, 1455, 1033,  793 }, ver = 127568, groups = { { 13,  3 }, {  2, 29 }, { 42,  1 }, { 23, 28 } }, codewords = { { { 115,  30 }, { 116,  30 } }, { {  46,  28 }, {  47,  28 } }, { {  24,  30 }, {  25,  30 } }, { {  15,  30 }, {  16,  30 } } }, align = {   6,  30,  56,  82, 108, 134       }, },
			{ capacity = { 1955, 1541, 1115,  845 }, ver = 133589, groups = { { 17,    }, { 10, 23 }, { 10, 35 }, { 19, 35 } }, codewords = { { { 115,  30 }               }, { {  46,  28 }, {  47,  28 } }, { {  24,  30 }, {  25,  30 } }, { {  15,  30 }, {  16,  30 } } }, align = {   6,  34,  60,  86, 112, 138       }, },
			{ capacity = { 2071, 1631, 1171,  901 }, ver = 136944, groups = { { 17,  1 }, { 14, 21 }, { 29, 19 }, { 11, 46 } }, codewords = { { { 115,  30 }, { 116,  30 } }, { {  46,  28 }, {  47,  28 } }, { {  24,  30 }, {  25,  30 } }, { {  15,  30 }, {  16,  30 } } }, align = {   6,  30,  58,  86, 114, 142       }, },
			{ capacity = { 2191, 1725, 1231,  961 }, ver = 141498, groups = { { 13,  6 }, { 14, 23 }, { 44,  7 }, { 59,  1 } }, codewords = { { { 115,  30 }, { 116,  30 } }, { {  46,  28 }, {  47,  28 } }, { {  24,  30 }, {  25,  30 } }, { {  16,  30 }, {  17,  30 } } }, align = {   6,  34,  62,  90, 118, 146       }, },
			{ capacity = { 2306, 1812, 1286,  986 }, ver = 145311, groups = { { 12,  7 }, { 12, 26 }, { 39, 14 }, { 22, 41 } }, codewords = { { { 121,  30 }, { 122,  30 } }, { {  47,  28 }, {  48,  28 } }, { {  24,  30 }, {  25,  30 } }, { {  15,  30 }, {  16,  30 } } }, align = {   6,  30,  54,  78, 102, 126, 150  }, },
			{ capacity = { 2434, 1914, 1354, 1054 }, ver = 150283, groups = { {  6, 14 }, {  6, 34 }, { 46, 10 }, {  2, 64 } }, codewords = { { { 121,  30 }, { 122,  30 } }, { {  47,  28 }, {  48,  28 } }, { {  24,  30 }, {  25,  30 } }, { {  15,  30 }, {  16,  30 } } }, align = {   6,  24,  50,  76, 102, 128, 154  }, },
			{ capacity = { 2566, 1992, 1426, 1096 }, ver = 152622, groups = { { 17,  4 }, { 29, 14 }, { 49, 10 }, { 24, 46 } }, codewords = { { { 122,  30 }, { 123,  30 } }, { {  46,  28 }, {  47,  28 } }, { {  24,  30 }, {  25,  30 } }, { {  15,  30 }, {  16,  30 } } }, align = {   6,  28,  54,  80, 106, 132, 158  }, },
			{ capacity = { 2702, 2102, 1502, 1142 }, ver = 158308, groups = { {  4, 18 }, { 13, 32 }, { 48, 14 }, { 42, 32 } }, codewords = { { { 122,  30 }, { 123,  30 } }, { {  46,  28 }, {  47,  28 } }, { {  24,  30 }, {  25,  30 } }, { {  15,  30 }, {  16,  30 } } }, align = {   6,  32,  58,  84, 110, 136, 162  }, },
			{ capacity = { 2812, 2216, 1582, 1222 }, ver = 161089, groups = { { 20,  4 }, { 40,  7 }, { 43, 22 }, { 10, 67 } }, codewords = { { { 117,  30 }, { 118,  30 } }, { {  47,  28 }, {  48,  28 } }, { {  24,  30 }, {  25,  30 } }, { {  15,  30 }, {  16,  30 } } }, align = {   6,  26,  54,  82, 110, 138, 166  }, },
			{ capacity = { 2956, 2334, 1666, 1276 }, ver = 167017, groups = { { 19,  6 }, { 18, 31 }, { 34, 34 }, { 20, 61 } }, codewords = { { { 118,  30 }, { 119,  30 } }, { {  47,  28 }, {  48,  28 } }, { {  24,  30 }, {  25,  30 } }, { {  15,  30 }, {  16,  30 } } }, align = {   6,  30,  58,  86, 114, 142, 170  }, }
		},
		maskInfo = {
			{ 30660, 29427, 32170, 30877, 26159, 25368, 27713, 26998 },
			{ 21522, 20773, 24188, 23371, 17913, 16590, 20375, 19104 },
			{ 13663, 12392, 16177, 14854,  9396,  8579, 11994, 11245 },
			{ 5769,   5054,  7399,  6608,  1890,   597,  3340,  2107 }
		},

		blank = "Qk1CAAAAAAAAAD4AAAAoAAAAAQAAAAEAAAABAAEAAAAAAAQAAAB0EgAAdBIAAAAAAAAAAAAAAAAAAP///wCAAAAA", -- 1x1px white bitmap.

		-- Generate the log & antilog tables for a galois finite field.
		galois = function(self, pp, size)
			local antilog, log, int = {[0] = 1}, {[0] = 1}, 2
			for power = 1, size - 1 do
				table.insert(antilog, int)
				log[int] = #antilog
				int = int * 2
				int = int > 0xff and ((int ~ pp) & 0xff) or int
				power = power + 1
			end
			return log, antilog
		end,

		-- Calculate the generator polynomials for sizes in supplied in 'list'. OK
		createGeneratorPolynomial = function(self, list)
			local function generator(poly, alpha)
				local alpha = {alpha, 0}
				local result = {}
				for x, a in ipairs(poly) do
					for X, A in ipairs(alpha) do
						local eA, eX = a + A, x + X - 2
						local newA, newX = (eA % 256) + math.floor(eA / 256), (eX % 256) + math.floor(eX / 256) + 1
						if result[newX] then
							result[newX] = self.log[self.antilog[result[newX]] ~ self.antilog[newA]]  -- If we already have a result for x, do galois addition.
						else
							result[newX] = newA
						end
					end
				end
				return result
			end

			-- Find the highest GP required.
			local last = 1
			for _, size in pairs(list) do
				last = last > size and last or size
			end

			-- Calculate all GP's up to 'last'.
			local gp = {{0, 0}}
			for i = 1, last - 1 do
				gp[i + 1] = generator(gp[i], i)
			end

			-- Return the requested generators.
			local result = {}
			for _, size in pairs(list) do
				result[size] = gp[size]
			end
			return result
		end,

		-- A processing queue for handling each chunk. Kind of works like a coroutine. OK
		q = {
			queue = {},

			tickInterval = 1,

			timer = Timer.New(),

			event = function(self, event)
				table.insert(self.queue, event)
				self.timer:Start(self.tickInterval)
			end,

			init = function(self, tickInterval)
				self.tickInterval = tickInterval
				self.timer.EventHandler = function(t)
					t:Stop()
					if #self.queue > 0 then
						if #self.queue > 1 then
							t:Start(self.tickInterval)
						end
						return table.remove(self.queue, 1)()
					else
						error("Queue empty. This call shouldn't happen.", 2)
					end
				end
			end
		},

		-- Scale a grid. Process in chunks. Callback is called with 'data' when complete. OK
		scale = function(self, data, factor, quiet, callback, state)
			if type(callback) ~= "function" then
				error("No callback supplied for scale!", 2)

			elseif not state then
				state = { quietSpace = #data[1], runCnt = 0 }
				state.data = {}
				for _, row in ipairs(data) do
					table.insert( state.data, table.pack(table.unpack(row)) )
				end
				return self.q:event(function() return self:scale(data, factor, quiet, callback, state) end) -- Queue the next step. Always queue first event.

			elseif state.quietSpace > 0 then  -- Add 'quiet' space around code.
				repeat
					for i = 1, quiet do
						table.insert(state.data[state.quietSpace], 1, 0)
						table.insert(state.data[state.quietSpace], 0)
					end
					state.quietSpace = state.quietSpace - 1
					state.runCnt = state.runCnt + 1
				until(state.quietSpace <= 0 or state.runCnt > self.loopCnt)
				state.runCnt = 0
				if state.quietSpace <= 0 then
					local blankRow = {}
					for i = 1, #state.data[1] do
						blankRow[i] = 0
					end
					for i = 1, quiet do
						table.insert(state.data, 1, table.pack(table.unpack(blankRow))) -- Insert a COPY of 'row'.
						table.insert(state.data, table.pack(table.unpack(blankRow))) -- Insert a COPY of 'row'.
						state.data[1].n = nil  -- Remove the stupid 'n' key that table.pack adds.
						state.data[#state.data].n = nil  -- Remove the stupid 'n' key that table.pack adds.
					end
					state.col = #state.data[1]
					state.colCnt = factor - 1
					state.row = #state.data
					state.rowCnt = factor - 1
				end
				return self.q:event(function() return self:scale(data, factor, quiet, callback, state) end) -- Queue the next step. Always queue first event.

			elseif state.col > 0 or state.row > 0 then
				repeat
					if state.col > 0 then -- Duplicate column pixel data then advance.
						state.data[state.row][state.col] = state.data[state.row][state.col] > 0 and 1 or 0  -- Clamp range to binary
						table.insert(state.data[state.row], state.col, state.data[state.row][state.col])
						state.colCnt = state.colCnt - 1
						if state.colCnt < 1 then
							state.colCnt = factor - 1
							state.col = state.col - 1
						end

					elseif state.row > 0 then -- Duplicate row then advance.
						table.insert(state.data, state.row, state.data[state.row])

						state.rowCnt = state.rowCnt - 1
						if state.rowCnt < 1 then
							state.rowCnt = factor - 1
							state.row = state.row - 1
							if state.row > 0 then
								state.col = #state.data[1]
							end
						end
					end
					state.runCnt = state.runCnt + 2
				until(state.row <= 0 or state.runCnt > self.loopCnt)
				state.runCnt = 0
				return self.q:event(function() return self:scale(data, factor, quiet, callback, state) end) -- Queue the next step. Always queue first event.

			else  -- We are done! Fire the callback.
				callback(state.data)
				state = nil -- Cleanup
			end
		end,

		-- Generate a printable grid. Useful for printing QR codes into the debug window.
		generateGridString = function(self, data, border, callback, state)
			if type(callback) ~= "function" then
				error("No callback supplied for generateGridString!", 2)

			elseif not state then
				state = { str = "", row = 1 }
				return self.q:event(function() return self:generateGridString(data, border, callback, state) end) -- Queue the next step. Always queue first event.

			elseif not state.firstLine then -- Generate top line for border, if required.
				if border then
					state.str = state.str .. "\r\n┌" .. ("──"):rep(#data + 4) .. "┐\r\n" .. ("│    " .. ("  "):rep(#data) .. "    │\r\n"):rep(2)
				else
					state.str = state.str .. "\r\n"
				end
				state.firstLine = true
				return self.q:event(function() return self:generateGridString(data, border, callback, state) end) -- Queue the next step.

			elseif state.row <= #data then  -- Generate the pixel blocks.
				if border then
					state.str = state.str .. "│    "
				end
				for _, pixel in ipairs(data[state.row]) do
					state.str = state.str .. (pixel > 0 and "██" or "  ")
				end
				if border then
					state.str = state.str .. "    │\r\n"
				else
					state.str = state.str .. "\r\n"
				end
				state.row = state.row + 1
				return self.q:event(function() return self:generateGridString(data, border, callback, state) end) -- Queue the next step.

			elseif border and not state.lastLine then -- Generate bottom line for border, if required.
				state.str = state.str .. ("│    " .. ("  "):rep(#data) .. "    │\r\n"):rep(2) .. "└" .. ("──"):rep(#data + 4) .. "┘\r\n"
				state.lastLine = true
				return self.q:event(function() return self:generateGridString(data, border, callback, state) end) -- Queue the next step.

			else  -- We are done! Fire the callback.
				callback(state.str)
				state = nil -- Cleanup
			end
		end,

		-- Generate a monochrome bitmap. Useful for putting QR codes on buttons. OK
		generateBitmap = function(self, data, callback, state)
			if type(callback) ~= "function" then
				error("No callback supplied for generateBitmap!", 2)

			elseif not state then
				state = { width = #data[1], height = #data, pixels = "", runCnt = 0}
				state.rowIdx = state.height
				return self.q:event(function() return self:generateBitmap(data, callback, state) end) -- Queue the next step. Always queue first event.

			elseif state.rowIdx > 0 then  -- Pack the pixel data into 4 byte boundaries.
				repeat
					for columnIdx = 1, state.width, 32 do
						local row = {table.unpack(data[state.rowIdx], columnIdx, columnIdx + 31)} -- Get 4 bytes of row data.
						while #row < 32 do table.insert(row, 0) end -- Pad to 4 byte boundary
						state.pixels = state.pixels .. bitstring.pack( ("1:int, "):rep(32), table.unpack(row) ) -- Pack the bits into 4 byte blocks.
						state.runCnt = state.runCnt + 1
					end
					state.rowIdx = state.rowIdx - 1
					state.runCnt = state.runCnt + 1
				until(state.rowIdx <= 0 or state.runCnt > self.loopCnt)
				state.runCnt = 0
				return self.q:event(function() return self:generateBitmap(data, callback, state) end) -- Queue the next step.

			elseif not state.bitmap then
				local header = string.pack( "<I4<I4<I4<I2<I2<I4<I4<I4<I4<I4<I4c8", 40, state.width, state.height, 1, 1, 0, 0, state.width, state.height, 2, 0, "\xff\xff\xff\x00\x00\x00\x00\x00")
				state.bitmap = string.pack("c2<I4xxxx<I4", "BM", (#header + 14 + #state.pixels), (#header + 14)) .. header .. state.pixels
				return self.q:event(function() return self:generateBitmap(data, callback, state) end) -- Queue the next step.

			else  -- We are done! Fire the callback.
				callback(state.bitmap)
				state = nil -- Cleanup
			end
		end,

		-- Calculate scaling factor for a desired horizontal resolution, then call scale() OK
		setRes = function(self, data, res, quiet, callback, state)
			local factor = math.floor(res / #data[1])
			return self:scale(data, factor, quiet, callback, state)
		end,

		-- Calculate the smallest version possible for the supplied string. OK
		calculateVersion = function( self, data, ec_level )
			local ec_level = ec_level and ec_level > 0 and ec_level < 5 and ec_level or 1  -- Ensure ec_level is between 1-4.
			for ec_levelValid = ec_level, 1, -1 do
				for version, versionCapabilites in ipairs(self.qrSpec) do
					if (versionCapabilites.capacity[ec_levelValid] - (version < 10 and 4 or 5)) >= #data then
						return version, ec_levelValid
					end
				end
			end
			return nil, ("Supplied string is too long. String must be smaller than 2951 characters. String was %d characters."):format(#data)
		end,

		-- Build an empty 'matrix' with the correct finder, alignment, and timing patterns.
		buildMatrix = function(self, groups, version, ec_level, callback, state)
			if type(callback) ~= "function" then
				error("No callback supplied for buildMatrix!", 2)

			elseif not state then -- Build the empty matrix.
				state = { matrix = {}, size = ((version - 1) * 4) + 21, dataCols = 0, ecCols = 0, group = 1, block = 1, col = 1, dataInserted = false, index = -1, odd = true, runCnt = 0, str = "\r\n"}
				-- Fnd out how many 'columns' we need to iterate through to interleave the data into the matrix.
				for _, t in ipairs(self.qrSpec[version].codewords[ec_level]) do
					state.dataCols = t[1] > state.dataCols and t[1] or state.dataCols
					state.ecCols = t[2] > state.ecCols and t[2] or state.ecCols
				end

				-- Add 'Finder' pattern.
				local function addFinder(x, y)
					-- Add pattern.
					for yI = 0, 6 do
						for xI = 0, 6 do
							state.matrix[y + yI][x + xI] = (yI == 0 or yI == 6 or xI == 0 or xI == 6 or (xI > 1 and xI < 5 and yI > 1 and yI < 5)) and 2 or -2
						end
					end
					-- Add separators.
					for p = 1, 8 do
						state.matrix[y == 1 and 8 or y - 1][x == 1 and p or (x + p - 2)] = -2
						state.matrix[y == 1 and p or (y + p - 2)][x == 1 and 8 or x - 1] = -2
					end
				end

				-- Add 'Alignment' pattern
				local function addAlign(x, y)
					-- Check that we can put a pattern here.
					for yI = 0, 4 do
						for xI = 0, 4 do
							if state.matrix[y + yI][x + xI] ~= 0 then -- If there is something in the matrix, skip this pattern.
								return
							end
						end
					end
					-- If we got this far, then write the pattern to the matrix.
					for yI = 0, 4 do
						for xI = 0, 4 do
							state.matrix[y + yI][x + xI] = (yI == 0 or yI == 4 or xI == 0 or xI == 4 or (xI == 2 and yI == 2)) and 2 or -2
						end
					end
				end

				local row = {}
				for i = 1, state.size do
					row[i] = 0
				end
				for i = 1, state.size do
					state.matrix[i] = table.pack(table.unpack(row)) -- Insert a COPY of 'row'.
					state.matrix[i].n = nil  -- Remove the stupid 'n' key that table.pack adds.
				end

				-- Add the finder patterns.
				addFinder(1, 1) -- Top Left
				addFinder(1, #state.matrix - 6) -- Bottom Left
				addFinder(#state.matrix - 6, 1) -- Top Right

				-- Add alignment patterns.
				for _, x in ipairs(self.qrSpec[version].align) do
					for _, y in ipairs(self.qrSpec[version].align) do
						addAlign(x - 1, y - 1)
					end
				end

				-- Reserve format information area. Some extra bits get set here, but they get tidied up by the timing pattern, and dark module.
				for m = 1, 8 do
					state.matrix[9][m] = -2
					state.matrix[m][9] = -2
					state.matrix[9][#state.matrix - m + 1] = -2
					state.matrix[#state.matrix - m + 1][9] = -2
				end
					state.matrix[9][9] = -2

				-- Reserve version information area. Only applies to version 7+.
				if version >= 7 then
					for a = 1, 6 do
						for b = 1, 3 do
							state.matrix[#state.matrix - b - 7][a] = -2
							state.matrix[a][#state.matrix - b - 7] = -2
						end
					end
				end

				-- Add timing patterns.
				for m = 9, state.size - 8 do
					state.matrix[7][m] = m % 2 == 1 and 2 or -2
					state.matrix[m][7] = m % 2 == 1 and 2 or -2
				end

				-- Add 'dark' module.
				state.matrix[#state.matrix - 7][9] = 2

				-- Set variables for next operation.
				state.y = #state.matrix
				state.x = state.y

				state.runCnt = state.runCnt + (self.dataEnable and 1 or 2) -- Skip data for debug.
				return self.q:event(function() return self:buildMatrix(groups, version, ec_level, callback, state) end) -- Queue the next step.

			elseif state.runCnt == 1 then -- Pack the data into the matrix.
				local loopCnt = 0

				local function insertBit(bit) -- Insert a single bit into the matrix.
					repeat  -- Repeat until break.
						if state.odd then
							state.odd = not state.odd
							if state.matrix[state.y][state.x] == 0 then
								state.matrix[state.y][state.x] = bit == 1 and 1 or -1
								break
							end
						else
							state.odd = not state.odd
							if state.matrix[state.y][state.x - 1] == 0 then
								state.matrix[state.y][state.x - 1] = bit == 1 and 1 or -1
								break
							end
							state.y = state.y + state.index

							if state.y < 1 or state.y > #state.matrix then
								state.index = -state.index
								state.y = state.y + state.index
								state.x = (state.x == 9) and (state.x - 3) or (state.x - 2)
							end
							if state.x < 1 then
								error("Too many data bits for matrix! This should never happen!")
							end
						end
						loopCnt = loopCnt + 1
					until(false)
				end

				local function insertByte(byte)
					for i = 7, 0, -1 do
						insertBit( (byte >> i) & 1 )
						loopCnt = loopCnt + 1
					end
				end

				repeat  -- Loop to the max loop count, then continue in the next control frame.
					if not state.dataInserted then  -- Interleave all the data blocks into the matrix.
						local byte = groups[state.group][state.block].data[state.col]
						if byte then
							insertByte(byte)
							state.str = state.str .. ("Data block from Group %d, Block %d, Column %d\t\t%d\r\n"):format(state.group, state.block, state.col, byte)
						end
						state.block = state.block + 1
						if state.block > #groups[state.group] then
							state.block = 1
							state.group = state.group + 1
							if state.group > #groups then
								state.group = 1
								state.col = state.col + 1
								if state.col > state.dataCols then
									-- Now do EC blocks.
									state.block = 1
									state.group = 1
									state.col = 1
									state.dataInserted = true
								end
							end
						end
					end
					if state.dataInserted then  -- Now interleave all the EC blocks.
						local byte = groups[state.group][state.block].ec[state.col]
						if byte then
							insertByte(byte)
							state.str = state.str .. ("EC block from Group %d, Block %d, Column %d\t\t%d\r\n"):format(state.group, state.block, state.col, byte)
						end
						state.block = state.block + 1
						if state.block > #groups[state.group] then
							state.block = 1
							state.group = state.group + 1
							if state.group > #groups then
								state.group = 1
								state.col = state.col + 1
								if state.col > state.ecCols then
									-- All done!
									state.runCnt = state.runCnt + 1
									break
								end
							end
						end
					end
					loopCnt = loopCnt + 1
				until(loopCnt > self.loopCnt)
				return self.q:event(function() return self:buildMatrix(groups, version, ec_level, callback, state) end) -- Queue the next step.

			else
				callback(state.matrix)  -- We are done! Fire the callback.
				state = nil -- Cleanup
			end
		end,

		-- Calculate the EC blocks for the supplied data blocks. OK
		calculateEC = function(self, data, n, callback, state)
			if type(callback) ~= "function" then
				error("No callback supplied for calculateEC!", 2)

			elseif not state then
				state = { data = table.pack(table.unpack(data)), firstTerm = self.log[data[1]], divStep = 1, expStep = 1, runCnt = 0}
				state.data.n = nil  -- Remove the stupid 'n' key that table.pack adds.
			return self.q:event(function() return self:calculateEC(data, n, callback, state) end) -- Queue the next step. Always queue first event.

			elseif state.divStep <= #data then  -- There are divisions left to do.
				repeat
					if state.data[1] == 0 and state.expStep == 1 then  -- Skip divide by zero on first term.
						table.remove(state.data, 1)
						table.insert(state.data, 0)
						state.divStep = state.divStep + 1
                      
						-- Prepare lead term for next multiply.
						state.firstTerm = self.log[state.data[1]]
					else
						-- Multiply the generator term by the lead message term.
						local alphaGP = self.generatorPolynomial[n][#self.generatorPolynomial[n] - state.expStep + 1] and (self.generatorPolynomial[n][#self.generatorPolynomial[n] - state.expStep + 1] + state.firstTerm) % 255 or -1


						-- XOR generator digit with the message digit.
						state.data[state.expStep] = (self.antilog[alphaGP] or 0) ~ (state.data[state.expStep] or 0)

						state.expStep = state.expStep + 1
						if state.expStep > #data and state.expStep > #self.generatorPolynomial[n] then

							-- Remove leading 0 terms.
							table.remove(state.data, 1)
							table.insert(state.data, 0)

							-- Prepare lead term for next multiply.
							state.firstTerm = self.log[state.data[1]]

							state.divStep = state.divStep + 1
							state.expStep = 1
						end
					end
					state.runCnt = state.runCnt + 1
				until(state.divStep > #data or state.runCnt > self.loopCnt)
				state.runCnt = 0
				return self.q:event(function() return self:calculateEC(data, n, callback, state) end) -- Queue the next step.

			else  -- We are done! Fire the callback.
				state.data = table.pack(table.unpack(state.data, 1, n)) -- Trim to the required number of EC Blocks.
				state.data.n = nil  -- Remove the stupid 'n' key that table.pack adds.
				callback(state.data)
				state = nil -- Cleanup
			end
		end,

		-- Calculate EC Blocks for each group. OK
		ecGroups = function( self, groups, callback, state )
			if type(callback) ~= "function" then
				error("No callback supplied for formatString!", 2)
			else
				for _, group in ipairs(groups) do
					for b, block in ipairs(group) do
						if type(block.ec) == "number" then
							return self:calculateEC( block.data, block.ec, function(ec)
								block.ec = ec
								return self:ecGroups( groups, callback )
							end)
						end
					end
				end
				callback(groups)  -- We are done! Fire the callback.
			end
		end,

		-- Add mode & length headers, and pad data to version size. OK
		formatData = function(self, data, version, ec_level, callback, state)
			local loopCnt = 0

			-- Insert a number of bits at the start of the table.
			local function insertHeaders(v, numBits)
				for i = 0, numBits - 1 do
					table.insert(state.bits, 1, (v >> i) & 1)
				end
				loopCnt = loopCnt + 1
			end

			-- Insert a number of bits at the end of the table.
			local function insertTerminators(v, numBits, prefix)
				for i = numBits - 1, 0, -1 do
					table.insert(state.bits, (v >> i) & 1)
				end
				loopCnt = loopCnt + 1
			end

			if type(callback) ~= "function" then
				error("No callback supplied for formatString!", 2)

			elseif not state then
				state = { bits = {}, currentChar = 1, runCnt = 0}
				state.bits = #data > 0 and table.pack( bitstring.unpack( ("1:int, "):rep(#data * 8), data ) ) or {}
				state.bits.n = nil  -- Remove the stupid 'n' key that table.pack adds.

				-- Insert headers. These are done in reverse.
				local sizeBits = version < 10 and 8 or 16
				insertHeaders((#data & ((2 ^ sizeBits) - 1)), sizeBits) -- Insert data size.
				insertHeaders(4, 4)  -- Byte mode
				insertHeaders(3, 8)  -- ECI type
				insertHeaders(7, 4)  -- ECI mode

				-- Append terminator, and pad to 8 bit boundary.
				insertTerminators(0, 4) -- Terminator.
				insertTerminators(0, 8 - (#state.bits % 8))  -- Pad to 8 bit multiple.
				return self.q:event(function() return self:formatData(data, version, ec_level, callback, state) end) -- Queue the next step. Always queue first event.

			elseif state.runCnt == 0 then -- Pad the string to the version/ec_level required length.
				repeat
					if #state.bits / 8 < self.qrSpec[version].capacity[ec_level] then
						insertTerminators( 236, 8 )
					end
					if #state.bits / 8 < self.qrSpec[version].capacity[ec_level] then
						insertTerminators( 17, 8 )
					end
					loopCnt = loopCnt + 1
				until(#state.bits / 8 >= self.qrSpec[version].capacity[ec_level] or loopCnt > self.loopCnt)
				if #state.bits / 8 > self.qrSpec[version].capacity[ec_level] then
					error(("The string is too long for the matrix! This should never happen! %d %d"):format(#state.bits, self.qrSpec[version].capacity[ec_level]) )
				elseif #state.bits / 8 == self.qrSpec[version].capacity[ec_level] then -- We have reached the correct length. We can process to the callback.
					state.data = bitstring.pack( ("1:int, "):rep(#state.bits), table.unpack(state.bits) ) -- Pack data back into a string.
					state.bits = nil  -- Done with bits.
					state.runCnt = state.runCnt + 1
				end
				return self.q:event(function() return self:formatData(data, version, ec_level, callback, state) end) -- Queue the next step.

			elseif state.runCnt == 1 then -- Split the string into the required data blocks.
				state.groups = {}
				state.position = 0
				for group = 1, #self.qrSpec[version].groups[ec_level] do
					state.groups[group] = {}
					local dataBlocks, ecBlocks = self.qrSpec[version].codewords[ec_level][group][1], self.qrSpec[version].codewords[ec_level][group][2]
					for block = 1, self.qrSpec[version].groups[ec_level][group] do
						state.groups[group][block] = {
							data = table.pack( string.unpack( ("B"):rep(dataBlocks), state.data:sub(state.position + 1, state.position + dataBlocks) ) ),
							ec = ecBlocks
						}
						state.groups[group][block].data.n = nil  -- Remove the stupid 'n' key that table.pack adds.
						table.remove(state.groups[group][block].data, #state.groups[group][block].data) -- Remove the stupid position indicator that string.unpack adds at the end.
						state.position = state.position + dataBlocks
					end
				end
				state.data = nil  -- Done with data string.
				state.runCnt = state.runCnt + 1
				return self.q:event(function() return self:formatData(data, version, ec_level, callback, state) end) -- Queue the next step.

			else  -- We are done! Fire the callback.
				callback(state.groups)
				state = nil -- Cleanup
			end
		end,

		addMask = function( self, matrix, mask, callback, state )
			if type(callback) ~= "function" then
				error("No callback supplied for addMask!", 2)

			elseif not state then
				state = {penalty = 0, darkModules = 0, x = 1, y = 1, xCnt = 0, yCnt = 0, runCnt = 1}
				state.matrix = {}
				for _, row in ipairs(matrix) do
					table.insert( state.matrix, table.pack(table.unpack(row)) )
					state.matrix[#state.matrix].n = nil  -- Remove the stupid 'n' key that table.pack adds.
				end
				return self.q:event(function() return self:addMask( matrix, mask, callback, state ) end) -- Queue the next step. Always queue first event.

			elseif state.runCnt == 1 then -- Apply appropriate mask.
				local loopCnt = 0
				repeat
					repeat
						local mod = state.matrix[state.y][state.x]
						if mod > -2 and mod < 2 and self.maskEnable then
							if mask == 0 and math.fmod((state.y - 1 + state.x - 1), 2) == 0 then
								state.matrix[state.y][state.x] = mod > 0 and 0 or 1
							elseif mask == 1 and math.fmod(state.y - 1, 2) == 0 then
								state.matrix[state.y][state.x] = mod > 0 and 0 or 1
							elseif mask == 2 and math.fmod(state.x - 1, 3) == 0 then
								state.matrix[state.y][state.x] = mod > 0 and 0 or 1
							elseif mask == 3 and math.fmod((state.y - 1 + state.x - 1), 3) == 0 then
								state.matrix[state.y][state.x] = mod > 0 and 0 or 1
							elseif mask == 4 and math.fmod((math.floor((state.y - 1)/2) + math.floor((state.x - 1)/3)), 2) == 0 then
								state.matrix[state.y][state.x] = mod > 0 and 0 or 1
							elseif mask == 5 and math.fmod(((state.y - 1) * (state.x - 1)), 2) + math.fmod(((state.y - 1) * (state.x - 1)), 3) == 0 then
								state.matrix[state.y][state.x] = mod > 0 and 0 or 1
							elseif mask == 6 and math.fmod( math.fmod(((state.y - 1) * (state.x - 1)), 2) + math.fmod(((state.y - 1) * (state.x - 1)), 3), 2 ) == 0 then
								state.matrix[state.y][state.x] = mod > 0 and 0 or 1
							elseif mask == 7 and math.fmod( math.fmod(((state.y - 1) + (state.x - 1)), 2) + math.fmod(((state.y - 1) * (state.x - 1)), 3), 2 ) == 0 then
								state.matrix[state.y][state.x] = mod > 0 and 0 or 1
							end
						end
						state.x = state.x + 1
						loopCnt = loopCnt + 1
					until(state.x > #state.matrix or loopCnt > self.loopCnt)
					if state.x > #state.matrix then
						state.x = 1
						state.y = state.y + 1
						if mask == 1 then state.y = state.y + 1 end -- Skip 1 row for mask 1. This saves a tiny bit of time.
					end
				until(state.y > #state.matrix or loopCnt > self.loopCnt)
				if state.y > #state.matrix then
					state.x = 1
					state.y = 1
					state.runCnt = state.runCnt + 1
				end
				return self.q:event(function() return self:addMask( matrix, mask, callback, state ) end) -- Queue the next step.

			elseif state.runCnt == 2 then -- Calculate penalty.
				local loopCnt = 0
				repeat
					repeat

					-- Check for 5 or more in a row.
					if ((state.matrix[state.y][state.x] or 0) > 0) == ((state.matrix[state.y][state.x + 1] or 0) > 0) then
						state.xCnt = state.xCnt + 1
						if state.xCnt == 4 then
						state.penalty = state.penalty + 3
						elseif state.xCnt > 4 then
						state.penalty = state.penalty + 1
						end
					else
						state.xCnt = 0
					end

					-- Check for 5 or more in a column.
					if ((state.matrix[state.x][state.y] or 0) > 0) == ((state.matrix[state.x + 1] and state.matrix[state.x + 1][state.y] or 0) > 0) then
						state.yCnt = state.yCnt + 1
						if state.yCnt == 4 then
						state.penalty = state.penalty + 3
						elseif state.yCnt > 4 then
						state.penalty = state.penalty + 1
						end
					else
						state.yCnt = 0
					end

					-- Check for same color blocks.
					if  ((state.matrix[state.y][state.x] or 0) > 0) == ((state.matrix[state.y][state.x + 1] or 0) > 0) and
						((state.matrix[state.y][state.x] or 0) > 0) == ((state.matrix[state.y + 1] and state.matrix[state.y + 1][state.x] or 0) > 0) and
						((state.matrix[state.y][state.x] or 0) > 0) == ((state.matrix[state.y + 1] and state.matrix[state.y + 1][state.x + 1] or 0) > 0) then
						state.penalty = state.penalty + 3
					end

					-- Check horizontally for finder lookalikes.
					if  (
							((state.matrix[state.y][state.x] or 0) > 0) == true and
							((state.matrix[state.y][state.x + 1] or 0) > 0) == false and
							((state.matrix[state.y][state.x + 2] or 0) > 0) == true and
							((state.matrix[state.y][state.x + 3] or 0) > 0) == true and
							((state.matrix[state.y][state.x + 4] or 0) > 0) == true and
							((state.matrix[state.y][state.x + 5] or 0) > 0) == false and
							((state.matrix[state.y][state.x + 6] or 0) > 0) == true
						)
						and
						(
							(
							state.matrix[state.y][state.x + 10] and
							((state.matrix[state.y][state.x + 7] or 0) > 0) == false and
							((state.matrix[state.y][state.x + 8] or 0) > 0) == false and
							((state.matrix[state.y][state.x + 9] or 0) > 0) == false and
							((state.matrix[state.y][state.x + 10] or 0) > 0) == false
							)
							or
							(
							state.matrix[state.y][state.x - 4] and
							((state.matrix[state.y][state.x - 1] or 0) > 0) == false and
							((state.matrix[state.y][state.x - 2] or 0) > 0) == false and
							((state.matrix[state.y][state.x - 3] or 0) > 0) == false and
							((state.matrix[state.y][state.x - 4] or 0) > 0) == false
							)
						)
					then
						state.penalty = state.penalty + 40
					end

					-- Check vertically for finder lookalikes.
					if  (
							state.matrix[state.x + 6] and
							((state.matrix[state.x][state.y] or 0) > 0) == true and
							((state.matrix[state.x + 1][state.y] or 0) > 0) == false and
							((state.matrix[state.x + 2][state.y] or 0) > 0) == true and
							((state.matrix[state.x + 3][state.y] or 0) > 0) == true and
							((state.matrix[state.x + 4][state.y] or 0) > 0) == true and
							((state.matrix[state.x + 5][state.y] or 0) > 0) == false and
							((state.matrix[state.x + 6][state.y] or 0) > 0) == true
						)
						and
						(
							(
							state.matrix[state.x + 10] and
							state.matrix[state.x + 10][state.y] and
							((state.matrix[state.x + 7][state.y] or 0) > 0) == false and
							((state.matrix[state.x + 8][state.y] or 0) > 0) == false and
							((state.matrix[state.x + 9][state.y] or 0) > 0) == false and
							((state.matrix[state.x + 10][state.y] or 0) > 0) == false
							)
							or
							(
							state.matrix[state.x - 4] and
							state.matrix[state.x - 4][state.y] and
							((state.matrix[state.x - 1][state.y] or 0) > 0) == false and
							((state.matrix[state.x - 2][state.y] or 0) > 0) == false and
							((state.matrix[state.x - 3][state.y] or 0) > 0) == false and
							((state.matrix[state.x - 4][state.y] or 0) > 0) == false
							)
						)
					then
						state.penalty = state.penalty + 40
					end

					-- Count 'dark' modules.
					state.darkModules = ((state.matrix[state.y][state.x] or 0) > 0) and state.darkModules + 1 or state.darkModules

					state.x = state.x + 1
					loopCnt = loopCnt + 4
					until(state.x > #state.matrix or loopCnt > self.loopCnt)
					state.x = 1
					state.y = state.y + 1
				until(state.y > #state.matrix or loopCnt > self.loopCnt)
				if state.y > #state.matrix then
					-- Add penalty for light/dark ratio.
					local lower, upper = math.abs((math.floor( (state.darkModules / ((#matrix) ^ 2) * 100) / 5 ) * 5) - 50) / 5, math.abs((math.ceil( (state.darkModules / ((#matrix) ^ 2) * 100) / 5 ) * 5) - 50) / 5
					if upper > lower then
					state.penalty = state.penalty + upper * 10
					else
					state.penalty = state.penalty + lower * 10
					end

					state.runCnt = state.runCnt + 1
				end
				return self.q:event(function() return self:addMask( matrix, mask, callback, state ) end) -- Queue the next step.

			else  -- We are done! Fire the callback.
				callback(state.matrix, state.penalty)
				state = nil -- Cleanup
			end
		end,

		getMatrix = function( self, groups, version, ec_level, callback, state )
			if type(callback) ~= "function" then
				error("No callback supplied for getMatrix!", 2)

			elseif not state then
				state = { matrixes = {} }
			end

			if #state.matrixes < 8 then
				self:buildMatrix( groups, version, ec_level, function( matrix )
					self:addMask( matrix, #state.matrixes, function( matrix, penalty )
					table.insert(state.matrixes, {matrix = matrix, penalty = penalty})
					return self.q:event(function() return self:getMatrix( groups, version, ec_level, callback, state ) end) -- Queue the next step.
					end)
				end)

			else  -- We are done! Fire the callback.
				-- Find the lowest penalty mask.
				local mask, penalty = 1, state.matrixes[1].penalty
				for idx, matrix in ipairs(state.matrixes) do
					if matrix.penalty < penalty then
						mask = idx
						penalty = matrix.penalty
					end
				end
				callback(self.maskOverride and state.matrixes[self.maskOverride].matrix or state.matrixes[mask].matrix, self.maskOverride or mask, penalty)
				state = nil -- Cleanup
			end
		end,

		addFormat = function( self, matrix, version, ec_level, mask, callback )
			-- Add version string.
			if version >= 7 then
				local x, y = 0, 0
				for i = 0, 17 do
					local bit = (self.qrSpec[version].ver >> i) & 1
					matrix[#matrix - 10 + y][x + 1] = bit
					matrix[x + 1][#matrix - 10 + y] = bit
					y = y + 1
					if y > 2 then
					x = x + 1
					y = 0
					end
				end
			end

			for i = 0, 14 do
				local bit = (self.maskInfo[ec_level][mask] >> (14 - i)) & 1
				matrix[9][i < 6 and i + 1 or (i < 7 and i + 2 or #matrix - 14 + i)] = bit
				matrix[i < 7 and #matrix - i or (i < 9 and 16 - i or 15 - i)][9] = bit
			end

			return self.q:event(function() return callback( matrix ) end) -- Queue the callback.
		end,

		buildQueue = {},
		buildRunning = false,
		queueBuild = function(self, build)
			if #self.buildQueue > self.maxBuilds then
				return "Build failed! Too many builds queued.", 2
			else
				table.insert( self.buildQueue, build )
				self:startBuild() -- Start the next build if we can.
				return "QR Code queued.", 5
			end
		end,
		startBuild = function(self)
			if #self.buildQueue > 0 and not self.buildRunning then
			self.buildRunning = true
			table.remove(self.buildQueue, 1)()
			end
		end,
		buildComplete = function(self)
			self.buildRunning = false
			self:startBuild()
		end,

		generate = function( self, t )
			if not type(t) == "table" then
				error("Invalid parameter table supplied.", 2)
			elseif not type(t.completedCallback) == "function" then
				error("No 'completedCallback' callback supplied.", 2)
			elseif type(t.ec_level) ~= "nil" and type(t.ec_level) == "number" and (t.ec_level > 4 or t.ec_level < 0) then
				error("Invalid EC Level supplied.", 2)
			else

			local version, ec_level = self:calculateVersion( t.string, t.ec_level and math.floor(t.ec_level) or 4 )

			if version == nil then
				if type(t.statusCallback) == "function" then
					t.statusCallback( ec_level, 2 )
				end

			else  -- Build the QR!
				buildStatusStr, buildStatusVal = self:queueBuild( function()

					local totalSteps = type(t.stringCallback) == "fuction" and 6 or 5
					version = ((type(t.version) == "number" and t.version > version) and t.version or version)  -- Use requested version if possible.

					if type(t.statusCallback) == "function" then t.statusCallback( ("1/%d Formatting data..."):format(totalSteps), 5 ) end
					return self:formatData( t.string, version, ec_level, function( groups )
						if type(t.statusCallback) == "function" then t.statusCallback( ("2/%d Calculating EC blocks..."):format(totalSteps), 5 ) end
						return self:ecGroups( groups, function( groups )
							if type(t.statusCallback) == "function" then t.statusCallback( ("3/%d Masking matrix..."):format(totalSteps), 5 ) end
							return self:getMatrix( groups, version, ec_level, function( matrix, mask, penalty )
								if type(t.statusCallback) == "function" then t.statusCallback( ("4/%d Scaling matrix..."):format(totalSteps), 5 ) end
								return self:addFormat( matrix, version, ec_level, mask, function( matrix )
									return self:setRes( matrix, 500, 2, function( scaledMatrix )
										if type(t.statusCallback) == "function" then t.statusCallback( ("5/%d Generating Bitmap..."):format(totalSteps), 5 ) end
										return self:generateBitmap( scaledMatrix, function( bitmap )

											if type(t.statusCallback) == "function" and type(t.stringCallback) ~= "function" then
												t.statusCallback( ("Done. Version: %d, EC Level: %d"):format(version, ec_level), 0 )
											end
											if type(t.stringCallback) ~= "function" then self:buildComplete() end  -- Start the next build if we can.
											t.completedCallback(bitmap, version, ec_level)

											if type(t.stringCallback) == "function" then
												if type(t.statusCallback) == "function" then t.statusCallback( ("6/%d Generating QR String..."):format(totalSteps), 5 ) end
												return self:generateGridString(matrix, true, function( string )
													if type(t.statusCallback) == "function" then
														t.statusCallback( ("Done. Version: %d, EC Level: %d"):format(version, ec_level), 0 )
													end
													self:buildComplete() -- Start the next build if we can.
													t.stringCallback(string)
												end)
											end

										end)
									end)
								end)
							end)
						end)
					end)
				end)
				if type(t.statusCallback) == "function" then
				t.statusCallback( buildStatusStr, buildStatusVal )
				end
			end

			end
		end,

		-- Perform initial setup operations.
		init = function(self, loopCnt, maxBuilds)
			self.loopCnt = loopCnt or 1000  -- How many loops to perform on each tick.
			self.maxBuilds = maxBuilds or 10  -- How many loops to perform on each tick.
			print("QR:\tBuilding log / antilog tables.")
			self.log, self.antilog = self:galois(285, 256) -- Build the log / antilog tables for PP=285 size=256.
			print("QR:\tBuilding generator polynomial tables.")
			self.generatorPolynomial = self:createGeneratorPolynomial({7, 10, 13, 15, 16, 17, 18, 20, 22, 24, 26, 28, 30}) -- Build a table of the required generator polynomials. We could do this inline when required.
			print("QR:\tInitialising the 'queue stack'.")
			self.q:init(0.001)  -- Setup the 'stack' queue.
			print( ("QR:\tReady. Running %d loops per frame."):format(loopCnt) )
		end,

		maskEnable = true,
		maskOverride = nil,
		dataEnable = true
	}

	cpu_levels = {
		Low = 1000,
		Medium = 2000,
		High = 3000
	}
	json = require("rapidjson")
	qr:init( cpu_levels[Properties["CPU Usage"].Value], Properties["QR Code Count"].Value * 2 ) -- 3000 is roughly the maximum safe value based on testing. Reducing the number reduces CPU load, but increases encode time.

	-- Populate the list of UCI's.
	local uciChoices = {}
	for i, v in pairs(json.decode(io.open("design/ucis.json","r"):read("*all")).Ucis) do
		uciChoices[i] = ("%d: %s"):format(i, v.Name) .. (v.IsPrivate and "(Private)" or "")
	end

	-- Find the nics & update the network interfaces comboboxes.
	local networkInterfaces = {}
	local networkCallbacks = {}
	local nicTimer = Timer.New()
	local function checkNics()
		nicTimer:Stop()
		networkInterfaces = {}
		for idx, nic in ipairs(Network.Interfaces()) do
			networkInterfaces[idx] =  ("%d: %s (%s)"):format(idx, nic.Interface, nic.Address)
		end
	for _, callback in ipairs(networkCallbacks) do callback() end
		nicTimer:Start(5)
	end
	nicTimer.EventHandler = checkNics

	-- Register controls for a page in the plugin.
	local function register( type_html_uci, type_iOS_uci, uci, nic, type_wifi, wifi_SSID, wifi_key, wifi_encryption, wifi_hidden, type_string, str, ec_level, version, image, status )
		local ec_levels = { "1: Low (7% Recovery)", "2: Medium (15% Recovery)", "3: Quartile (25% Recovery)", "4: High (30% Recovery)" }
		local encryptionTypes = {"WPA", "WEP"}
		local versions = { "Auto (Smallest Possible)" }
		for i = 1, 40 do
			local size = ((i - 1) * 4) + 21
			local maxChars = qr.qrSpec[i].capacity[1] - (i < 10 and 4 or 5)
			table.insert( versions, ("Version %d (%sx%d %d chars max.)"):format(i, size, size, maxChars) )
		end

		local function clear()
			image.Legend = '{"DrawChrome":false,"IconData":"' .. qr.blank .. '"}'
		end

		local buildTimer = Timer.New()
		buildTimer.EventHandler = function()
			buildTimer:Stop()
			local string = ""
			if type_html_uci.Boolean then
				string = HttpClient.CreateUrl{
					Host = ("http://%s"):format( nic.String:match("%((.+)%)$") ),
					Path = "uci-viewer/",
					Query = {
					file = ("%d.UCI.xml"):format( uci.String:match("^(%d+):") or 0 ),
					uci = uci.String:match("^%d+: (.+)"),
					directory = "/designs/current_design/UCIs/"
					}
				}
			elseif type_iOS_uci.Boolean then
				string = HttpClient.CreateUrl{
					Host = "qsys://",
					Query = {
					design = Design.GetStatus().DesignName,
					page = uci.String:match("^%d+: (.+)")
					}
				}
			elseif type_wifi.Boolean then
				string = ("WIFI:S:%s;T:%s;P:%s;H:%s;"):format(wifi_SSID.String:gsub("[:;]", "?"), wifi_encryption.String, wifi_key.String:gsub("[:;]", "?"), wifi_hidden.Boolean and "true" or "false")
			elseif type_string.Boolean then
				string = str.String
			end

			qr:generate{
				string = string,
				statusCallback = function(str, val)
					status.Value = val
					status.String = str
					if val == 2 then clear() end  -- Clear QR code if error.
				end,
				completedCallback = function(bitmap)
					image.Legend = '{"DrawChrome":false,"IconData":"' .. Crypto.Base64Encode(bitmap) .. '"}'
					print( ("Generated QR Code for string \"%s\""):format(string) )
				end,
				version = tonumber( version.String:match("^Version (%d+)") ),
				ec_level = ec_level.String:match("^(%d+):")
			}
		end

		local function update()
			buildTimer:Start(0.5)
		end

		local function inList(item, list) -- Check if 'item' is present in 'list'.
			for _, listItem in ipairs(list) do
				if item ==  listItem then return true end
			end
			return false
		end

		local function findRelated( ctl, list, trig )
			if ctl.IsDisabled then
				ctl.Color = ""
			elseif ctl.String:len() < 1 then
				ctl.Color = "red"
			else
				ctl.Color = "red"
				local name = ctl.String:match("%d+: ([^%(]+)")
				for idx, val in ipairs( list ) do
					if val:match("%d+: ([^%(]+)") == name then
						ctl.Color = ""
						if trig or ctl.String ~= list[idx] then
							ctl.String = list[idx]
							return update() -- Update if value changed.
						end
						break
					end
				end
			end
		end

		local function enable(list) -- Enable all listed control, disable others.
			local allCtls = { uci, nic, wifi_SSID, wifi_key, wifi_encryption, wifi_hidden, str }
			for _, ctl in ipairs(allCtls) do
				local disable = inList(ctl, list) == false
				ctl.IsDisabled = disable and true or false

				if disable then
					ctl.Color = ""
				elseif ctl == nic then
					findRelated( nic, networkInterfaces )
				end
			end
		end

		local function qrTypeSelect(...)
			local t = {...}
			local activeCtl
			local function doRadio(trig)
				for _, ctl in ipairs(t) do
					ctl.Boolean = ctl == trig
					ctl.Legend = ctl == trig and "✔" or ""
				end
				if type_html_uci.Boolean then
					enable{ uci, nic }
				elseif type_iOS_uci.Boolean then
					enable{ uci }
				elseif type_wifi.Boolean then
					enable{ wifi_SSID, wifi_key, wifi_encryption, wifi_hidden }
				elseif type_string.Boolean then
					enable{ str }
				end
				update()  -- Update the qr code!
			end
			for _, ctl in ipairs(t) do
				ctl.EventHandler = doRadio
				activeCtl = (type(activeCtl) == "nil" and ctl.Boolean) and ctl or activeCtl
			end
			doRadio( type(activeCtl) == "nil" and t[1] or activeCtl )  -- Ensure we have a valid selection.
		end

		local function isPresent(value, table)
			for _, v in pairs(table) do
				if v ==  value then
					return true
				end
			end
			return false
		end

		for _, ctl in ipairs({ wifi_SSID, wifi_key, wifi_encryption, wifi_hidden, str, ec_level, version }) do
			ctl.EventHandler = update
		end

		uci.Choices = uciChoices
		uci.EventHandler = function( ctl ) findRelated( ctl, uciChoices, true ) end
		findRelated( uci, uciChoices )

		nic.Choices = networkInterfaces
		nic.EventHandler = function( ctl ) findRelated( ctl, networkInterfaces, true ) end
		findRelated( nic, networkInterfaces )

		ec_level.Choices = ec_levels
		ec_level.String = isPresent(ec_level.String, ec_levels) and ec_level.String or ec_levels[3]

		version.Choices = versions
		version.String = isPresent(version.String, versions) and version.String or versions[1]

		wifi_encryption.Choices = encryptionTypes
		wifi_encryption.String = isPresent(wifi_encryption.String, encryptionTypes) and wifi_encryption.String or "WPA"

		qrTypeSelect( type_html_uci, type_iOS_uci, type_wifi, type_string )

		clear()
		return function()
			nic.Choices = networkInterfaces
			findRelated( nic, networkInterfaces )
		end
	end

	checkNics()	-- Start polling the NICs.

	-- Register all the controls.
	if Properties["QR Code Count"].Value > 1 then
		for i = 1, Properties["QR Code Count"].Value do
			table.insert( networkCallbacks, register(
				Controls.type_html_uci[i],
				Controls.type_iOS_uci[i],
					Controls.uci[i],
					Controls.nic[i],
				Controls.type_wifi[i],
					Controls.wifi_SSID[i],
					Controls.wifi_key[i],
					Controls.wifi_encryption[i],
					Controls.wifi_hidden[i],
				Controls.type_string[i],
					Controls.str[i],
				Controls.ec_level[i],
				Controls.version[i],
				Controls.image[i],
				Controls.status[i]
			))
		end
	else
		table.insert( networkCallbacks, register(
			Controls.type_html_uci,
			Controls.type_iOS_uci,
				Controls.uci,
				Controls.nic,
			Controls.type_wifi,
				Controls.wifi_SSID,
				Controls.wifi_key,
				Controls.wifi_encryption,
				Controls.wifi_hidden,
			Controls.type_string,
				Controls.str,
			Controls.ec_level,
			Controls.version,
			Controls.image,
			Controls.status
		))
	end

end

--[[
DISCLAIMER:
THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE
LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR
CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF
SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS
INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN
CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE)
ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE
POSSIBILITY OF SUCH DAMAGE.
--]]
